Обновил `README.md` под ваш актуальный `rotator.py` (с автоматическим переключением не только ключей при `429`, но и резервных моделей при `503`/`500`, а также с общим таймаутом `25` секунд).

> Важная деталь: так как скрипт открывает `keys.txt` по относительному пути (`open("keys.txt")`), запускать его нужно из папки `~/gemini-rotator`, а в `systemd`-сервисе обязательно указать `WorkingDirectory=%h/gemini-rotator` (добавил это в инструкцию).

---

```markdown
# Gemini API Proxy, Key Rotator & Model Fallback для VS Code (Cline)

Локальный прокси-ротатор на Python (без сторонних библиотек), который:
1. Направляет запросы к `generativelanguage.googleapis.com` через внешний HTTP-прокси (`100.64.0.101:8888`).
2. Автоматически перебирает API-ключи из `keys.txt` при ошибке лимита (`429`).
3. Автоматически переключается на резервную модель из списка `MODELS` при перегрузке (`503` / `500`) или сетевом сбое.
4. Ограничивает общее время попыток на один запрос (`25` секунд), чтобы Cline не зависал надолго.

---

## Схема работы

`VS Code (Cline)` ──▶ `rotator.py (127.0.0.1:4000)` ──▶ `HTTP-прокси (100.64.0.101:8888)` ──▶ `generativelanguage.googleapis.com`

---

## 1. Проверка внешнего прокси

Перед настройкой проверьте на новом ПК, что прокси `100.64.0.101:8888` доступен и отдаёт поддерживаемую страну (например, `US`, `NL`, `DE`, а не `BY`/`RU`):

```bash
curl -x http://100.64.0.101:8888 -s ipinfo.io/country && echo ""
```

---

## 2. Установка на новом ПК (Ubuntu / Linux)

Создайте папку:

```bash
mkdir -p ~/gemini-rotator && cd ~/gemini-rotator
```

### 2.1. Создайте файл со списком API-ключей `keys.txt`

```bash
nano ~/gemini-rotator/keys.txt
```

Вставьте ключи по одному на строку, например:
```text
AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXX1
AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXX2
AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXX3
```

Сохранение в `nano`: `Ctrl + O` → `Enter` → `Ctrl + X`.

### 2.2. Создайте скрипт `rotator.py`

```bash
cat << 'EOF' > ~/gemini-rotator/rotator.py
import http.server, urllib.request, urllib.error, itertools, re, time

PROXY = "http://100.64.0.101:8888"
KEYS = [l.strip() for l in open("keys.txt") if l.strip()]
MODELS = ["gemini-4-hydrogen", "gemini-3.8-flash", "gemini-3.6-flash", "gemini-3.5-flash", "gemini-3.5-flash-lite", "gemini-2.5-flash"]
cycle = itertools.cycle(KEYS)
cur = next(cycle)
opener = urllib.request.build_opener(urllib.request.ProxyHandler({"http": PROXY, "https": PROXY}))

class H(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        global cur
        start = time.time()
        body = self.rfile.read(int(self.headers.get("Content-Length", 0)))
        clean_p = re.sub(r'([?&])key=[^&]*&?', r'\1', self.path).rstrip('?&')
        m = re.search(r'/models/([^:]+)', clean_p)
        req_m = m.group(1) if m else ""
        try_models = [req_m] + [x for x in MODELS if x != req_m] if req_m else [""]
        dom = "generativelanguage" + "." + "googleapis" + "." + "com"

        for model in try_models:
            if time.time() - start > 25:
                break
            path = re.sub(r'/models/[^:]+', f'/models/{model}', clean_p) if model else clean_p
            for i in range(len(KEYS)):
                if time.time() - start > 25:
                    break
                u = f"https://{dom}{path}{'&' if '?' in path else '?'}key={cur}"
                req = urllib.request.Request(u, data=body, method="POST")
                req.add_header("Content-Type", self.headers.get("Content-Type", "application/json"))
                req.add_header("x-goog-api-key", cur)
                try:
                    with opener.open(req, timeout=20) as r:
                        print(f"[OK] {model} | ключ {cur[:10]}...")
                        self.send_response(r.status)
                        self.send_header("Content-Type", r.headers.get("Content-Type", "application/json"))
                        self.end_headers()
                        self.wfile.write(r.read())
                        return
                except urllib.error.HTTPError as e:
                    err = e.read()
                    if e.code == 429:
                        cur = next(cycle)
                        continue
                    if e.code in (503, 500):
                        print(f"[503] {model} занята -> следующая")
                        break
                    self.send_response(e.code)
                    self.end_headers()
                    self.wfile.write(err)
                    return
                except Exception:
                    break

        self.send_response(503)
        self.end_headers()
        self.wfile.write(b'{"error": {"message": "Google Free Tier busy, retry"}}')

print(f"Ротатор готов (ключей: {len(KEYS)}, моделей: {len(MODELS)})")
http.server.ThreadingHTTPServer(("127.0.0.1", 4000), H).serve_forever()
EOF
```

### 2.3. Ручной запуск (для проверки)

Запускать нужно обязательно из папки `~/gemini-rotator`:

```bash
cd ~/gemini-rotator && python3 rotator.py
```

---

## 3. Автозапуск в фоне через systemd (опционально)

Чтобы ротатор стартовал сам при входе в систему:

```bash
mkdir -p ~/.config/systemd/user

cat << 'EOF' > ~/.config/systemd/user/gemini-rotator.service
[Unit]
Description=Gemini API Key & Model Rotator
After=network-online.target

[Service]
WorkingDirectory=%h/gemini-rotator
ExecStart=/usr/bin/python3 %h/gemini-rotator/rotator.py
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now gemini-rotator.service
```

Проверка статуса и логов в реальном времени:

```bash
systemctl --user status gemini-rotator.service
journalctl --user -u gemini-rotator.service -f
```

---

## 4. Настройка VS Code и Cline

### 4.1. Настройки Cline
Откройте настройки Cline (шестерёнка):
* **API Provider:** `Google Gemini`
* **Gemini API Key:** любое слово (например, `rotator` — скрипт всё равно заменит его на ключ из `keys.txt`)
* **Use custom base URL:** включено (`✓`)
* **Custom Base URL:** `http://127.0.0.1:4000`
* **Model:** `gemini-4-hydrogen` (при `503`/`500` скрипт сам откатится по цепочке: `gemini-3.8-flash` → `gemini-3.6-flash` → `gemini-3.5-flash` → `gemini-3.5-flash-lite` → `gemini-2.5-flash`)

### 4.2. Настройки Proxy в VS Code (важный нюанс)
* Если прокси `100.64.0.101:8888` нужен **только для работы Gemini в Cline**, в самом VS Code прокси настраивать не обязательно — `rotator.py` сам ходит через `100.64.0.101:8888`.
* Если в VS Code (`Ctrl + ,`) всё же прописаны `Http: Proxy` = `http://100.64.0.101:8888` и `Http: Proxy Support` = `override`, добавьте `127.0.0.1` и `localhost` в **Http: No Proxy** (`settings.json`), чтобы запросы к локальному ротатору не уходили во внешний прокси:
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
   и перезапустите скрипт / сервис (`systemctl --user restart gemini-rotator.service`).
2. Если включён `ufw`, откройте порт `4000` для вашей локальной подсети:
   ```bash
   sudo ufw allow from 192.168.0.0/16 to any port 4000 proto tcp
   ```
3. Узнайте локальный IP главного ПК:
   ```bash
   hostname -I
   ```
4. На остальных ПК в сети в настройках Cline укажите:
   * **Use custom base URL:** `http://<IP_ГЛАВНОГО_ПК>:4000`
   * **Gemini API Key:** `rotator`
   * (Если в VS Code на клиентском ПК включён `http.proxy`, добавьте `<IP_ГЛАВНОГО_ПК>` в `http.noProxy`.)
```
