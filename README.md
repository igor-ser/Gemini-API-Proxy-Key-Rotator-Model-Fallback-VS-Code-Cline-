
# Gemini API Proxy, Key Rotator & Model Fallback для VS Code (Cline)

Локальный прокси-ротатор на Python (без сторонних библиотек), который:
1. Направляет запросы к `generativelanguage.googleapis.com` через внешний HTTP-прокси (`100.64.0.101:8888`).
2. Автоматически перебирает API-ключи из `keys.txt` при ошибке лимита (`429`).
3. Автоматически переключается на резервные модели из списка `MODELS` при перегрузке (`500`, `502`, `503`, `504`) или сетевом сбое, а при `404` исключает недоступную модель для текущего запроса.
4. Повторяет попытки по кругу с нарастающей паузой (`3с → 6с → 9с`) до `120` секунд (`MAX_WAIT`), чтобы не приходилось постоянно нажимать **Retry** в Cline.

---

## Схема работы

`VS Code (Cline)` ──▶ `rotator.py (127.0.0.1:4000)` ──▶ `HTTP-прокси (100.64.0.101:8888)` ──▶ `generativelanguage.googleapis.com`

---

## 1. Проверка внешнего прокси

Перед настройкой проверьте, что прокси `100.64.0.101:8888` доступен и отдаёт поддерживаемую страну (например, `US`, `NL`, `DE`, а не `BY`/`RU`):

```bash
curl -x http://100.64.0.101:8888 -s ipinfo.io/country && echo ""
```

---

## 2. Установка (Ubuntu / Linux)

Создайте рабочую папку:

```bash
mkdir -p ~/gemini-rotator && cd ~/gemini-rotator
```

### 2.1. Создайте файл со списком API-ключей `keys.txt`

```bash
nano ~/gemini-rotator/keys.txt
```

Вставьте ключи по одному на строку:
```text
AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXX1
AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXX2
AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXX3
```

Сохранение в `nano`: `Ctrl + O` → `Enter` → `Ctrl + X`.

### 2.2. Создайте скрипт `rotator.py`

```bash
cat << 'EOF' > ~/gemini-rotator/rotator.py
import http.server, urllib.request, urllib.error, itertools, re, time, threading, sys

sys.stdout.reconfigure(line_buffering=True)

PROXY = "http://100.64.0.101:8888"
KEYS = [l.strip() for l in open("keys.txt") if l.strip()]
MODELS = [
    "gemini-4-hydrogen",
    "gemini-3.8-flash",
    "gemini-3.6-flash",
    "gemini-3.5-flash",
    "gemini-3.5-flash-lite",
    "gemini-2.5-flash",
]
MAX_WAIT = 120      # сколько секунд максимум пытаться достучаться до Google
REQ_TIMEOUT = 15    # таймаут одной попытки (сек)
RETRY_DELAY = 3     # базовая пауза между кругами (растёт 3с -> 6с -> 9с)

lock = threading.Lock()
cycle = itertools.cycle(KEYS)
cur = next(cycle)
opener = urllib.request.build_opener(urllib.request.ProxyHandler({"http": PROXY, "https": PROXY}))

def get_key():
    with lock:
        return cur

def next_key():
    global cur
    with lock:
        cur = next(cycle)
        return cur

class H(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        start = time.time()
        body = self.rfile.read(int(self.headers.get("Content-Length", 0)))
        clean_p = re.sub(r'([?&])key=[^&]*&?', r'\1', self.path).rstrip('?&')
        m = re.search(r'/models/([^:]+)', clean_p)
        req_m = m.group(1) if m else ""

        # Сначала модель из Cline, потом запасные из MODELS.
        # Если хотите ВСЕГДА начинать с первой модели из MODELS (gemini-4-hydrogen)
        # независимо от настроек Cline, замените строку ниже на:
        # try_models = list(MODELS) if req_m else [""]
        try_models = [req_m] + [x for x in MODELS if x != req_m] if req_m else [""]

        dom = "generativelanguage" + "." + "googleapis" + "." + "com"
        attempt = 0

        while time.time() - start < MAX_WAIT and try_models:
            attempt += 1
            dead_models = []
            key = get_key()

            for model in list(try_models):
                if time.time() - start >= MAX_WAIT:
                    break
                path = re.sub(r'/models/[^:]+', f'/models/{model}', clean_p) if model else clean_p

                for _ in range(len(KEYS)):
                    if time.time() - start >= MAX_WAIT:
                        break
                    u = f"https://{dom}{path}{'&' if '?' in path else '?'}key={key}"
                    req = urllib.request.Request(u, data=body, method="POST")
                    req.add_header("Content-Type", self.headers.get("Content-Type", "application/json"))
                    req.add_header("x-goog-api-key", key)
                    try:
                        with opener.open(req, timeout=REQ_TIMEOUT) as r:
                            print(f"[OK] {model} | ключ {key[:10]}... | круг #{attempt}")
                            self.send_response(r.status)
                            self.send_header("Content-Type", r.headers.get("Content-Type", "application/json"))
                            self.end_headers()
                            self.wfile.write(r.read())
                            return
                    except urllib.error.HTTPError as e:
                        err = e.read()
                        if e.code == 429:
                            print(f"[429] {model} | ключ {key[:10]}... -> следующий ключ")
                            key = next_key()
                            continue
                        if e.code in (500, 502, 503, 504):
                            print(f"[{e.code}] {model} занята -> следующая модель")
                            break
                        if e.code == 404:
                            print(f"[404] {model} не найдена -> убираю из списка")
                            dead_models.append(model)
                            break
                        self.send_response(e.code)
                        self.end_headers()
                        self.wfile.write(err)
                        return
                    except Exception as e:
                        print(f"[ERR] {model}: {e} -> следующая модель")
                        break

            for dm in dead_models:
                if dm in try_models:
                    try_models.remove(dm)

            left = MAX_WAIT - (time.time() - start)
            if left > 0 and try_models:
                delay = min(RETRY_DELAY * min(attempt, 3), left)
                print(f"[RETRY #{attempt}] Все модели/ключи заняты, жду {delay:.1f}с...")
                time.sleep(delay)

        self.send_response(503)
        self.end_headers()
        self.wfile.write(b'{"error": {"message": "Google Free Tier busy, retry"}}')

print(f"Ротатор готов (ключей: {len(KEYS)}, моделей: {len(MODELS)}, макс. ожидание: {MAX_WAIT}с)")
http.server.ThreadingHTTPServer(("127.0.0.1", 4000), H).serve_forever()
EOF
```

### 2.3. Ручной запуск (для проверки)

Запускать нужно из папки `~/gemini-rotator` (так как `keys.txt` открывается по относительному пути):

```bash
cd ~/gemini-rotator && python3 rotator.py
```

---

## 3. Автозапуск в фоне через systemd

```bash
mkdir -p ~/.config/systemd/user

cat << 'EOF' > ~/.config/systemd/user/gemini-rotator.service
[Unit]
Description=Gemini API Key & Model Rotator
After=network-online.target

[Service]
WorkingDirectory=%h/gemini-rotator
ExecStart=/usr/bin/python3 -u %h/gemini-rotator/rotator.py
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now gemini-rotator.service
```

Полезные команды:

```bash
# Статус сервиса
systemctl --user status gemini-rotator.service

# Логи в реальном времени (будет видно [OK], [429], [503], [RETRY])
journalctl --user -u gemini-rotator.service -f

# Перезапуск после правки rotator.py или keys.txt
systemctl --user restart gemini-rotator.service
```

---

## 4. Настройка VS Code и Cline

### 4.1. Настройки Cline
Откройте настройки Cline (шестерёнка):
* **API Provider:** `Google Gemini`
* **Gemini API Key:** любое слово (например, `rotator` — скрипт сам подставит ключ из `keys.txt`)
* **Use custom base URL:** включено (`✓`)
* **Custom Base URL:** `http://127.0.0.1:4000`
* **Model:** выберите нужную стартовую модель (например, `gemini-4-hydrogen` или `gemini-3.8-flash`).
  > **Важно:** ротатор начинает с той модели, которая указана в Cline (если только вы не включили `try_models = list(MODELS)` в коде скрипта — тогда приоритет всегда задаётся списком `MODELS`).

### 4.2. Настройки Proxy в VS Code
* Если прокси `100.64.0.101:8888` нужен **только для Cline**, в самом VS Code прокси включать не нужно — `rotator.py` ходит через него сам.
* Если в VS Code (`Ctrl + ,`) уже прописаны `Http: Proxy` = `http://100.64.0.101:8888` и `Http: Proxy Support` = `override`, обязательно добавьте `127.0.0.1` и `localhost` в `settings.json`, чтобы запросы к ротатору не уходили во внешний прокси:
  ```json
  "http.noProxy": [
    "127.0.0.1",
    "localhost"
  ]
  ```

---

## 5. Вариант для локальной сети (один ротатор на все ПК)

Чтобы не держать `keys.txt` и скрипт на каждом компьютере:

1. На главном ПК в последней строке `rotator.py` замените `"127.0.0.1"` на `"0.0.0.0"`:
   ```python
   http.server.ThreadingHTTPServer(("0.0.0.0", 4000), H).serve_forever()
   ```
   и перезапустите сервис:
   ```bash
   systemctl --user restart gemini-rotator.service
   ```
2. Если включён `ufw`, откройте порт `4000` для локальной сети:
   ```bash
   sudo ufw allow from 192.168.0.0/16 to any port 4000 proto tcp
   ```
3. Узнайте локальный IP главного ПК:
   ```bash
   hostname -I
   ```
4. На остальных ПК в настройках Cline укажите:
   * **Use custom base URL:** `http://<IP_ГЛАВНОГО_ПК>:4000`
   * **Gemini API Key:** `rotator`
   * (Если в VS Code на клиентском ПК включён `http.proxy`, добавьте `<IP_ГЛАВНОГО_ПК>` в `http.noProxy`.)

---

## 6. Тюнинг параметров в `rotator.py`

* `MAX_WAIT = 120` — максимальное время ожидания ответа (в секундах). Если Free Tier перегружен дольше 2 минут, можно увеличить до `180` или `300`.
* `REQ_TIMEOUT = 15` — сколько секунд ждать ответа от одной попытки, прежде чем перейти к следующей модели.
* `RETRY_DELAY = 3` — базовая пауза между кругами (увеличивается на 2-м и 3-м круге: `3с → 6с → 9с`).
* `MODELS` — порядок резервных моделей (сверху вниз).
```
