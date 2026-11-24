# Gemini API Proxy, Smart Key Rotator & Auto Model Fallback для VS Code (Cline)

Локальный прокси-ротатор на Python (без сторонних библиотек), который:
1. Направляет запросы к `generativelanguage.googleapis.com` через внешний HTTP-прокси (`100.64.0.101:8888`).
2. **Анализирует ошибки и автоматически меняет модель:**
   - При `429` определяет тип лимита (`PerMinute` / `retryDelay` или дневной `PerDay` / `RPD`) и замораживает пару `(ключ, модель)` на нужное время.
   - Если `KEYS_PER_MODEL` (по умолчанию 3) ключей подряд на модели дали `429`, **сама модель** автоматически ставится на паузу (`90с` при минутном лимите или `300с` при дневном RPD) — следующие запросы из Cline **мгновенно** идут сразу на рабочую модель, минуя проблемные. По истечении паузы скрипт сам проверит модель снова.
   - При `500 / 502 / 503 / 504` (перегрузка в часы пик США) модель сразу уходит на паузу `90с`, а запрос уходит на следующую по списку.
   - При `404` модель автоматически исключается из списка до перезапуска сервиса.
3. **Защищает IP от бана:** делает анти-спам паузу (`0.5с`) между неудачными ключами и ограничивает число попыток на одну модель за круг.

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

Вставьте ключи по одному на строку (пустые строки и строки, начинающиеся с `#`, игнорируются):

```text
# Лучше по 1 ключу из разных Google-аккаунтов/проектов
AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXX1
AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXX2
AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXX3
```

> **Важно:** если добавить 30–50 ключей из **одного** Google Cloud проекта, квота у них будет общая. Лучше держать по 1 ключу на каждый независимый Google-аккаунт/проект.

Сохранение в `nano`: `Ctrl + O` → `Enter` → `Ctrl + X`.

### 2.2. Создайте скрипт `rotator.py`

```bash
cat << 'EOF' > ~/gemini-rotator/rotator.py
import http.server, urllib.request, urllib.error, re, time, threading, sys, os, json, random

sys.stdout.reconfigure(line_buffering=True)

PROXY = "http://100.64.0.101:8888"
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
KEYS = [
    l.strip()
    for l in open(os.path.join(BASE_DIR, "keys.txt"), encoding="utf-8")
    if l.strip() and not l.strip().startswith("#")
]

if not KEYS:
    raise SystemExit("Ошибка: файл keys.txt пуст!")

MODELS = [
    "gemini-4-hydrogen",
    "gemini-3.8-flash",
    "gemini-3.6-flash",
    "gemini-3.5-flash",
    "gemini-3.5-flash-lite",
    "gemini-2.5-flash",
]

MAX_WAIT = 120            # макс. время ожидания ответа (сек)
REQ_TIMEOUT = 45          # таймаут одного запроса к Google (сек)
RETRY_DELAY = 5           # пауза между кругами, если все модели заняты (сек)
KEY_DELAY = 0.5           # анти-спам пауза при смене ключа после 429 (сек)
KEYS_PER_MODEL = 3        # макс. попыток ключей на одну модель за круг

MODEL_RPM_COOLDOWN = 90   # пауза модели, если подряд сработал RPM лимит (сек)
MODEL_RPD_COOLDOWN = 300  # пауза модели, если подряд сработал дневной RPD (сек)
MODEL_5XX_COOLDOWN = 90   # пауза модели при перегрузке 500/502/503/504 (сек)

lock = threading.Lock()
key_idx = 0
key_cooldowns = {}        # (key, model) -> timestamp
model_cooldowns = {}      # model -> timestamp
dead_models_global = set()

opener = urllib.request.build_opener(
    urllib.request.ProxyHandler({"http": PROXY, "https": PROXY})
)

def pick_key(model):
    global key_idx
    now = time.time()
    with lock:
        n = len(KEYS)
        for offset in range(n):
            pos = (key_idx + offset) % n
            k = KEYS[pos]
            if key_cooldowns.get((k, model), 0) <= now:
                key_idx = (pos + 1) % n
                return k
    return None

def set_key_cooldown(key, model, seconds):
    with lock:
        key_cooldowns[(key, model)] = time.time() + seconds

def set_model_cooldown(model, seconds):
    if not model:
        return
    with lock:
        model_cooldowns[model] = max(model_cooldowns.get(model, 0), time.time() + seconds)

def get_ordered_models(req_m):
    now = time.time()
    with lock:
        base = [req_m] + [x for x in MODELS if x != req_m] if req_m else [""]
        candidates = [m for m in base if m not in dead_models_global]
        ready = [m for m in candidates if model_cooldowns.get(m, 0) <= now]
        paused = sorted(
            [m for m in candidates if model_cooldowns.get(m, 0) > now],
            key=lambda m: model_cooldowns[m],
        )
    return ready, paused

def parse_429(err_bytes):
    text = err_bytes.decode("utf-8", errors="ignore")
    delay = 60
    is_daily = False
    reason = "rate limit"
    try:
        data = json.loads(text)
        msg = data.get("error", {}).get("message", "")
        if msg:
            reason = msg.split(".")[0][:80]
        for d in data.get("error", {}).get("details", []):
            if "retryDelay" in d:
                m = re.search(r'(\d+)', str(d["retryDelay"]))
                if m:
                    delay = max(10, int(m.group(1)))
    except Exception:
        pass
    if "PerDay" in text or "daily" in text.lower():
        delay = 1800  # дневная квота -> заморозка пары (ключ, модель) на 30 мин
        is_daily = True
        reason = "Daily quota (RPD)"
    return delay, is_daily, reason

class H(http.server.BaseHTTPRequestHandler):
    def log_message(self, format, *args):
        pass

    def do_POST(self):
        start = time.time()
        body = self.rfile.read(int(self.headers.get("Content-Length", 0)))
        clean_p = re.sub(r'([?&])key=[^&]*&?', r'\1', self.path).rstrip('?&')
        m = re.search(r'/models/([^:]+)', clean_p)
        req_m = m.group(1) if m else ""

        dom = "generativelanguage" + "." + "googleapis" + "." + "com"
        attempt = 0

        while time.time() - start < MAX_WAIT:
            attempt += 1
            ready_models, paused_models = get_ordered_models(req_m)
            if not ready_models and not paused_models:
                break

            # Если есть активные модели — работаем с ними; если все на паузе — пробуем с наименьшей паузой
            models_to_try = ready_models if ready_models else paused_models
            if not ready_models:
                print(f"[INFO] Все модели в паузе, пробую резервный обход...")

            for model in models_to_try:
                if time.time() - start >= MAX_WAIT:
                    break

                path = re.sub(r'/models/[^:]+', f'/models/{model}', clean_p) if model else clean_p
                tries_for_model = min(KEYS_PER_MODEL, len(KEYS))
                all_429 = True
                all_daily = True
                got_429 = False

                for _ in range(tries_for_model):
                    if time.time() - start >= MAX_WAIT:
                        break

                    key = pick_key(model)
                    if not key:
                        print(f"[SKIP] {model}: все ключи на паузе -> следующая модель")
                        break

                    u = f"https://{dom}{path}{'&' if '?' in path else '?'}key={key}"
                    req = urllib.request.Request(u, data=body, method="POST")
                    req.add_header("Content-Type", self.headers.get("Content-Type", "application/json"))
                    req.add_header("x-goog-api-key", key)

                    try:
                        with opener.open(req, timeout=REQ_TIMEOUT) as r:
                            resp_data = r.read()
                            ctype = r.headers.get("Content-Type", "application/json")
                            status = r.status
                        print(f"[OK] {model} | ключ {key[:10]}... | круг #{attempt}")
                        self.send_response(status)
                        self.send_header("Content-Type", ctype)
                        self.end_headers()
                        self.wfile.write(resp_data)
                        return
                    except urllib.error.HTTPError as e:
                        err = e.read()
                        if e.code == 429:
                            got_429 = True
                            cd, is_daily, reason = parse_429(err)
                            all_daily = all_daily and is_daily
                            set_key_cooldown(key, model, cd)
                            print(f"[429] {model} | {key[:10]}... | пауза {cd}с ({reason})")
                            time.sleep(KEY_DELAY + random.uniform(0.1, 0.3))
                            continue
                        all_429 = False
                        if e.code in (500, 502, 503, 504):
                            set_model_cooldown(model, MODEL_5XX_COOLDOWN)
                            print(f"[{e.code}] {model} занята -> пауза модели {MODEL_5XX_COOLDOWN}с")
                            break
                        if e.code == 404:
                            with lock:
                                dead_models_global.add(model)
                            print(f"[404] {model} не найдена -> отключена")
                            break
                        self.send_response(e.code)
                        self.end_headers()
                        self.wfile.write(err)
                        return
                    except Exception as e:
                        all_429 = False
                        print(f"[ERR] {model}: {e} -> следующая модель")
                        break
                else:
                    # Если все tries_for_model попыток на модели закончились 429 — ставим саму модель на паузу
                    if got_429 and all_429 and model:
                        m_cd = MODEL_RPD_COOLDOWN if all_daily else MODEL_RPM_COOLDOWN
                        set_model_cooldown(model, m_cd)
                        print(f"[MODEL PAUSE] {model} -> пауза {m_cd}с, дальше пойдём сразу на другие")

            left = MAX_WAIT - (time.time() - start)
            if left > 0:
                delay = min(RETRY_DELAY * min(attempt, 3), left)
                print(f"[RETRY #{attempt}] Все модели заняты, жду {delay:.1f}с...")
                time.sleep(delay)

        self.send_response(503)
        self.end_headers()
        self.wfile.write(b'{"error": {"message": "Google Free Tier busy, retry"}}')

print(f"Ротатор готов (ключей: {len(KEYS)}, моделей: {len(MODELS)}, макс. ожидание: {MAX_WAIT}с)")
http.server.ThreadingHTTPServer(("127.0.0.1", 4000), H).serve_forever()
EOF
```

### 2.3. Ручной запуск и проверка ключа

Для ручного запуска:
```bash
cd ~/gemini-rotator && python3 rotator.py
```

Для ручной проверки конкретного ключа через `curl` (подставьте реальный ключ из `keys.txt` латиницей):
```bash
KEY=$(head -n 1 ~/gemini-rotator/keys.txt)
curl -s -x http://100.64.0.101:8888 \
  "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=$KEY" \
  -H "Content-Type: application/json" \
  -d '{"contents":[{"parts":[{"text":"hi"}]}]}'
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

# Логи в реальном времени
journalctl --user -u gemini-rotator.service -f

# Перезапуск после правки rotator.py или keys.txt (сбрасывает паузы моделей и 404)
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
* **Model:** выберите желаемую стартовую модель (например, `gemini-4-hydrogen` или `gemini-3.5-flash`).
  > Если стартовая модель упрётся в лимиты/перегрузку, ротатор сам временно отправит её на паузу и следующие запросы из Cline будет сразу направлять на рабочую модель из `MODELS`.

### 4.2. Настройки Proxy в VS Code
* Если прокси `100.64.0.101:8888` нужен **только для Cline**, в самом VS Code прокси включать не нужно — `rotator.py` ходит через него сам.
* Если в VS Code (`Ctrl + ,`) уже прописаны `Http: Proxy` = `http://100.64.0.101:8888` и `Http: Proxy Support` = `override`, обязательно добавьте локальные адреса в `settings.json`:
  ```json
  "http.noProxy": [
    "127.0.0.1",
    "localhost"
  ]
  ```

---

## 5. Вариант для локальной сети (один ротатор на все ПК)

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
   * *(Если в VS Code на клиентском ПК включён `http.proxy`, добавьте `<IP_ГЛАВНОГО_ПК>` в `http.noProxy`.)*

---

## 6. Параметры и расшифровка логов

### Основные параметры в `rotator.py`
* `MODELS` — приоритет моделей сверху вниз.
* `KEYS_PER_MODEL = 3` — сколько разных ключей пробовать на одной модели, прежде чем сменить модель (защита от спама).
* `KEY_DELAY = 0.5` — пауза между ключами при `429`.
* `MODEL_RPM_COOLDOWN = 90` — на сколько секунд пропускать модель при минутном лимите.
* `MODEL_RPD_COOLDOWN = 300` — на сколько секунд пропускать модель при дневном лимите (`RPD`).
* `MODEL_5XX_COOLDOWN = 90` — на сколько секунд пропускать модель при ошибке перегрузки `503`.
* `REQ_TIMEOUT = 45` — таймаут ожидания ответа от Google (увеличен до 45с, чтобы длинные ответы Cline не обрывались).
* `MAX_WAIT = 120` — общее максимальное время попыток на один запрос.

### Что означают сообщения в `journalctl`
* `[OK] ...` — запрос успешно выполнен.
* `[429] ... | пауза Xс (...)` — ключ упёрся в лимит и заморожен для этой модели на `X` секунд.
* `[MODEL PAUSE] ...` — на модели сработали лимиты на всех проверенных ключах подряд; следующие запросы будут сразу идти в обход неё.
* `[503] ... занята` — серверы Google перегружены (часто в рабочие часы США), модель поставлена на паузу `90с`.
* `[404] ... отключена` — модель не найдена по этому API-пути и отключена до перезапуска.
* `[SKIP] ...` — все ключи для данной модели сейчас на паузе, мгновенный переход к следующей.
* `[RETRY #N]` — все доступные модели заняты, ожидание перед следующим кругом.
