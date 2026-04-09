# Защищённый мессенджер своими руками: Matrix + Element

Полное руководство с E2E-шифрованием, звонками и обходом блокировок.

Свой мессенджер с полным шифрованием, голосовыми/видео звонками, без привязки к номеру телефона и без зависимости от чужих серверов. Идеально подходит для задач, где важна приватность и контроль над данными.

## Архитектура

- **Synapse** — сервер Matrix (Python, слушает на localhost:8008)
- **PostgreSQL** — база данных
- **Element Web** — веб-клиент (статика, раздаётся nginx)
- **nginx** — reverse proxy + SSL termination
- **Coturn** — TURN/STUN сервер для голосовых/видео звонков

## Что понадобится

- VPS с Ubuntu 22.04 или выше (минимум 1 vCPU, 2 ГБ RAM, 20 ГБ SSD)
- Домен (например `example.com`)
- Доступ к DNS-записям домена

## Шаг 0 — DNS

Создай три A-записи, указывающие на IP твоего VPS:

```
example.com         →  <IP сервера>
matrix.example.com  →  <IP сервера>
element.example.com →  <IP сервера>
```

TTL поставь 300 (5 минут) — это ускорит переключение DNS, если понадобится.

Проверка:

```bash
ping example.com
ping matrix.example.com
ping element.example.com
```

---

## Шаг 1 — Подготовка сервера

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg2 lsb-release apt-transport-https \
  ca-certificates nginx certbot python3-certbot-nginx \
  postgresql postgresql-contrib ufw
```

Настройка firewall:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 3478/tcp        # Coturn
sudo ufw allow 3478/udp
sudo ufw allow 5349/tcp        # Coturn TLS
sudo ufw allow 5349/udp
sudo ufw allow 49152:65535/udp # Coturn media relay
sudo ufw enable
```

---

## Шаг 2 — PostgreSQL

```bash
sudo -u postgres psql
```

В psql:

```sql
CREATE USER synapse_user WITH PASSWORD 'ПАРОЛЬ_БД';
CREATE DATABASE synapse
  ENCODING 'UTF8'
  LC_COLLATE='C'
  LC_CTYPE='C'
  template=template0
  OWNER synapse_user;
\q
```

> `LC_COLLATE='C'` обязателен — Synapse требует это для корректной сортировки.

---

## Шаг 3 — Установка Synapse

Добавляем официальный репозиторий Matrix.org:

При установке:
- **Server name**: вводи `example.com` (не `matrix.example.com`!) — это будет часть Matrix ID: `@user:example.com`
- **Report statistics**: выбирай **No**

```bash
sudo curl -fsSL https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/matrix-org-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] \
  https://packages.matrix.org/debian/ $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/matrix-org.list

sudo apt update
sudo apt install -y matrix-synapse-py3
```

Остановим для настройки:

```bash
sudo systemctl stop matrix-synapse
```

---

## Шаг 4 — Настройка homeserver.yaml

Бэкап оригинала:

```bash
sudo cp /etc/matrix-synapse/homeserver.yaml /etc/matrix-synapse/homeserver.yaml.bak
```

Генерируем секреты (запиши все три):

```bash
# Секрет для токенов
python3 -c "import secrets; print(secrets.token_hex(32))"

# Секрет для регистрации пользователей
python3 -c "import secrets; print(secrets.token_hex(32))"

# Секрет для Coturn
python3 -c "import secrets; print(secrets.token_hex(32))"
```

Открываем конфиг:

```bash
sudo nano /etc/matrix-synapse/homeserver.yaml
```

Удаляем всё содержимое и вставляем:

```yaml
pid_file: "/var/run/matrix-synapse.pid"

listeners:
  - bind_addresses:
      - 127.0.0.1
    port: 8008
    resources:
      - names:
          - client
        compress: false
    tls: false
    type: http
    x_forwarded: true

database:
  name: psycopg2
  args:
    user: synapse_user
    password: "ПАРОЛЬ_БД"
    database: synapse
    host: localhost
    cp_min: 5
    cp_max: 10

log_config: "/etc/matrix-synapse/log.yaml"
media_store_path: /var/lib/matrix-synapse/media
signing_key_path: "/etc/matrix-synapse/homeserver.signing.key"

# Открытая регистрация пользователей
enable_registration: true
enable_registration_without_verification: true

# Секрет для создания пользователей через CLI
registration_shared_secret: "СЕКРЕТ_РЕГИСТРАЦИИ"

# Отключаем федерацию — закрытый сервер
federation_domain_whitelist: []

# Подавляем предупреждение о key server
suppress_key_server_warning: true
trusted_key_servers: []

# Секрет для токенов авторизации
macaroon_secret_key: "СЕКРЕТ_ТОКЕНОВ"

# TURN-сервер для звонков
turn_uris:
  - "turn:matrix.example.com:3478?transport=udp"
  - "turn:matrix.example.com:3478?transport=tcp"
  - "turns:matrix.example.com:5349?transport=udp"
  - "turns:matrix.example.com:5349?transport=tcp"
turn_shared_secret: "СЕКРЕТ_COTURN"
turn_user_lifetime: 86400000
turn_allow_guests: false
```

> **Замени** все плейсхолдеры (ПАРОЛЬ_БД, СЕКРЕТ_РЕГИСТРАЦИИ, СЕКРЕТ_ТОКЕНОВ, СЕКРЕТ_COTURN) на реальные значения.

Сохраняем: `Ctrl+O`, `Enter`, `Ctrl+X`.

Проверь server_name:

```bash
cat /etc/matrix-synapse/conf.d/server_name.yaml
# Должно быть: server_name: "example.com"
```

---

## Шаг 5 — SSL-сертификаты

```bash
sudo certbot certonly --nginx -d matrix.example.com -d element.example.com
sudo certbot certonly --nginx -d example.com
```

Проверка автопродления:

```bash
sudo certbot renew --dry-run
```

---

## Шаг 6 — nginx

Создаём конфиг:

```bash
sudo nano /etc/nginx/sites-available/matrix
```

Вставляем:

```nginx
# Synapse — клиентский API
server {
    listen 443 ssl http2;
    server_name matrix.example.com;

    ssl_certificate /etc/letsencrypt/live/matrix.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.example.com/privkey.pem;

    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;
    }
}

# Element Web — фронтенд
server {
    listen 443 ssl http2;
    server_name element.example.com;

    ssl_certificate /etc/letsencrypt/live/matrix.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.example.com/privkey.pem;

    root /var/www/element;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}

# Основной домен — .well-known для обнаружения сервера
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location /.well-known/matrix/client {
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
        return 200 '{"m.homeserver": {"base_url": "https://matrix.example.com"}}';
    }

    location /.well-known/matrix/server {
        default_type application/json;
        return 200 '{"m.server": "matrix.example.com:443"}';
    }

    location / {
        return 404;
    }
}

# HTTP → HTTPS редирект
server {
    listen 80;
    server_name matrix.example.com element.example.com example.com;
    return 301 https://$host$request_uri;
}
```

Сохраняем: `Ctrl+O`, `Enter`, `Ctrl+X`.

Активируем:

```bash
sudo ln -s /etc/nginx/sites-available/matrix /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

---

## Шаг 7 — Запуск Synapse

```bash
sudo systemctl start matrix-synapse
sudo systemctl status matrix-synapse
```

Проверка:

```bash
curl https://matrix.example.com/_matrix/client/versions
```

Должен вернуть JSON со списком версий.

---

## Шаг 8 — Element Web

Скачиваем и разворачиваем:

```bash
cd /tmp
curl -sL https://github.com/element-hq/element-web/releases/download/v1.11.96/element-v1.11.96.tar.gz -o element.tar.gz
sudo mkdir -p /var/www/element
sudo tar -xzf element.tar.gz -C /var/www/element --strip-components=1
```

Создаём конфиг:

```bash
sudo nano /var/www/element/config.json
```

Вставляем:

```json
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://matrix.example.com",
            "server_name": "example.com"
        }
    },
    "disable_guests": true,
    "disable_3pid_login": true,
    "brand": "Example Messenger",
    "features": {
        "feature_group_calls": false
    },
    "element_call": {
        "url": ""
    },
    "voip": {
        "obey_asserted_identity": false
    }
}
```

Сохраняем: `Ctrl+O`, `Enter`, `Ctrl+X`.

Проверка:

```bash
curl -sI https://element.example.com | head -5
# Должен вернуть HTTP/2 200
```

---

## Шаг 9 — Coturn

```bash
sudo apt install -y coturn
```

Включаем автозапуск:

```bash
sudo sed -i 's/#TURNSERVER_ENABLED=1/TURNSERVER_ENABLED=1/' /etc/default/coturn
```

Создаём конфиг:

```bash
sudo nano /etc/turnserver.conf
```

Удаляем всё содержимое и вставляем:

```ini
listening-port=3478
tls-listening-port=5349
listening-ip=0.0.0.0

realm=matrix.example.com
server-name=matrix.example.com

use-auth-secret
static-auth-secret=СЕКРЕТ_COTURN

cert=/etc/letsencrypt/live/matrix.example.com/fullchain.pem
pkey=/etc/letsencrypt/live/matrix.example.com/privkey.pem

no-tcp-relay
denied-peer-ip=10.0.0.0-10.255.255.255
denied-peer-ip=192.168.0.0-192.168.255.255
denied-peer-ip=172.16.0.0-172.31.255.255

no-multicast-peers
no-cli
no-tlsv1
no-tlsv1_1
```

> **СЕКРЕТ_COTURN** — тот же, что в homeserver.yaml в `turn_shared_secret`.

Сохраняем: `Ctrl+O`, `Enter`, `Ctrl+X`.

Запуск:

```bash
sudo systemctl restart coturn
sudo systemctl status coturn
```

---

## Шаг 10 — Создание пользователей

```bash
register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml http://localhost:8008
```

- **Username**: имя пользователя (будет `@имя:example.com`)
- **Password**: надёжный пароль
- **Admin**: `yes` для первого пользователя (администратора)

Повторить для каждого пользователя.

---

## Шаг 11 — Вход и настройка

### В браузере

1. Открыть `https://element.example.com`
2. Зарегистрироваться
3. Настроить **Security Key** (Settings → Security & Privacy → Secure Backup) — **обязательно сохранить ключ!**
4. Установить Display Name и аватарку (Settings → General)

### На Android / iOS

1. Установить **Element** из Google Play / App Store
2. Sign in → Edit (рядом с `matrix.org`) → ввести `example.com`
3. Зарегистрироваться
4. Верифицировать новую сессию через браузер (сравнение эмодзи)

### Создание комнат

1. Нажать «+» рядом с Rooms → New Room
2. Включить шифрование (Encrypted)
3. Пригласить пользователей по Matrix ID: `@имя:example.com`

### Верификация между пользователями

В зашифрованной комнате нажать на имя собеседника → Verify → сравнить эмодзи на экранах.

---

## Проверки после установки

```bash
# Synapse работает
sudo systemctl status matrix-synapse

# Coturn работает
sudo systemctl status coturn

# nginx работает
sudo systemctl status nginx

# API отвечает
curl https://matrix.example.com/_matrix/client/versions

# .well-known отвечает
curl https://example.com/.well-known/matrix/client

# Element доступен
curl -sI https://element.example.com | head -3
```

---

## Убираем логи

```bash
sudo nano /etc/matrix-synapse/log.yaml
```

Выставляем параметр:

```yaml
root:
    level: CRITICAL
```

Перезагружаем Synapse:

```bash
sudo systemctl restart matrix-synapse
```

---

## Администрирование

Открой в браузере `https://admin.etke.cc`

В поле Homeserver URL введи `https://matrix.example.com`

Залогинься под админским аккаунтом.

---

## Шаг 12 — Бэкапы

Создаём папку и скрипт:

```bash
sudo mkdir -p /home/backup
```

```bash
sudo nano /home/backup/backup.sh
```

Вставляем:

```bash
#!/bin/bash

BACKUP_DIR="/home/backup"
DATE=$(date +%Y%m%d_%H%M)

# Бэкап базы данных
cd /tmp && sudo -u postgres pg_dump synapse | gzip > "$BACKUP_DIR/synapse_db_$DATE.sql.gz"

# Бэкап конфигов
tar -czf "$BACKUP_DIR/synapse_config_$DATE.tar.gz" \
  /etc/matrix-synapse/ \
  /etc/nginx/sites-available/ \
  /etc/turnserver.conf \
  /var/www/element/config.json \
  2>/dev/null

# Удаляем бэкапы старше 7 дней
find "$BACKUP_DIR" -name "synapse_*.gz" -mtime +7 -delete

echo "Backup done: $DATE"
```

Сохраняем: `Ctrl+O`, `Enter`, `Ctrl+X`.

Делаем исполняемым и проверяем:

```bash
sudo chmod +x /home/backup/backup.sh
sudo /home/backup/backup.sh
ls -la /home/backup/
```

Настраиваем ежедневный запуск в 4 утра:

```bash
sudo crontab -e
```

Выбираем редактор nano (1), добавляем строку в конец:

```
0 4 * * * /home/backup/backup.sh
```

Сохраняем: `Ctrl+O`, `Enter`, `Ctrl+X`.

Проверяем:

```bash
sudo crontab -l
```

Восстановление из бэкапа (при необходимости):

```bash
gunzip -c /home/backup/synapse_db_ДАТА.sql.gz | sudo -u postgres psql synapse
```

---

## Шаг 13 — Автообновление безопасности

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

Выбери **Yes** — система будет автоматически ставить security-обновления.

---

## Шаг 14 — Настройка прокси для белых списков

Запускаем сервер на Yandex Cloud.

Устанавливаем нужные пакеты:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx certbot python3-certbot-nginx libnginx-mod-stream ufw
```

Настраиваем firewall (SSH первым!):

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 3478/tcp
sudo ufw allow 3478/udp
sudo ufw allow 5349/tcp
sudo ufw allow 5349/udp
sudo ufw enable
```

Меняем A-записи в DNS (TTL 300):

```
matrix.example.com  →  <IP прокси>
element.example.com →  <IP прокси>
```

> A-запись `example.com` остаётся на IP основного сервера — она отдаёт `.well-known`.

SSL-сертификаты:

```bash
sudo certbot certonly --nginx -d matrix.example.com -d element.example.com
```

Проверка автопродления:

```bash
sudo certbot renew --dry-run
```

Настраиваем nginx:

```bash
sudo nano /etc/nginx/sites-available/matrix-relay
```

Вставляем:

```nginx
server {
    listen 443 ssl http2;
    server_name matrix.example.com;

    ssl_certificate /etc/letsencrypt/live/matrix.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.example.com/privkey.pem;

    client_max_body_size 50M;

    location / {
        proxy_pass https://<IP основного сервера>;
        proxy_set_header Host matrix.example.com;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl http2;
    server_name element.example.com;

    ssl_certificate /etc/letsencrypt/live/matrix.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.example.com/privkey.pem;

    location / {
        proxy_pass https://<IP основного сервера>;
        proxy_set_header Host element.example.com;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name matrix.example.com element.example.com;
    return 301 https://$host$request_uri;
}
```

Сохраняем: `Ctrl+O`, `Enter`, `Ctrl+X`.

Активируем:

```bash
sudo ln -s /etc/nginx/sites-available/matrix-relay /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Проверяем:

```bash
curl https://matrix.example.com/_matrix/client/versions
```

Проксирование Coturn для звонков:

```bash
sudo nano /etc/nginx/nginx.conf
```

Добавляем в самый конец файла, после закрывающей `}` блока `http`:

```nginx
stream {
    server {
        listen 3478;
        proxy_pass <IP основного сервера>:3478;
    }
    server {
        listen 3478 udp;
        proxy_pass <IP основного сервера>:3478;
    }
    server {
        listen 5349;
        proxy_pass <IP основного сервера>:5349;
    }
    server {
        listen 5349 udp;
        proxy_pass <IP основного сервера>:5349;
    }
}
```

Сохраняем: `Ctrl+O`, `Enter`, `Ctrl+X`.

Проверяем и перезагружаем:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## Клиенты Matrix

Matrix — это открытый протокол, и подключиться к серверу можно с любого совместимого клиента. Вот проверенные варианты:

### Рекомендуемые клиенты

| Клиент | Платформы | Особенности |
|--------|-----------|-------------|
| **Element Web** | Браузер | Основной веб-клиент, ставится на сервер вместе с Synapse |
| **Element Desktop** | Windows, macOS, Linux | Десктопное приложение на базе Electron |
| **Element Classic** | Android, iOS | Мобильный клиент, стабильный, поддерживает классические звонки через Coturn |
| **SchildiChat** | Android, Windows, macOS, Linux, Web | Форк Element с более привычным интерфейсом в стиле Telegram |
| **FluffyChat** | Android, iOS, Linux, Web | Лёгкий и красивый клиент, хорош для начинающих |
| **Cinny** | Windows, macOS, Linux, Web | Минималистичный клиент с современным интерфейсом в стиле Discord/Slack |

### Другие клиенты

| Клиент | Платформы | Описание |
|--------|-----------|----------|
| **Nheko** | Windows, macOS, Linux | Быстрый десктопный клиент на Qt/C++ |
| **NeoChat** | Windows, Linux | Клиент для KDE Plasma |
| **Fractal** | Linux | Клиент для GNOME |
| **Thunderbird** | Windows, macOS, Linux | Почтовый клиент с поддержкой Matrix |
| **gomuks** | macOS, Linux | Терминальный клиент на Go |
| **iamb** | Windows, macOS, Linux | Терминальный клиент с Vim-клавишами |

### Что учитывать при выборе клиента

При подключении к самостоятельному серверу в настройках клиента нужно указать адрес Homeserver: `https://matrix.example.com` (или свой домен, если настроен `.well-known`).

**Element X** (iOS, Android) — новое поколение мобильного клиента от разработчиков Element. Требует Matrix Authentication Service (MAS) для авторизации и LiveKit для звонков. На self-hosted серверах с базовой настройкой Synapse **не будет работать** для звонков и может не работать для регистрации. Для переписки — работает.

**SchildiChat Next** (Android) — форк Element X с поддержкой Spaces. Те же ограничения, что у Element X.

Полный каталог клиентов: https://matrix.org/ecosystem/clients/

---

## Преимущества перед другими мессенджерами

### По сравнению с Telegram
Telegram шифрует только «секретные чаты» один на один, и только на мобильных. Обычные и групповые чаты хранятся на серверах Telegram в открытом виде. На self-hosted Matrix все комнаты зашифрованы, включая групповые. Нет привязки к номеру телефона.

### По сравнению с WhatsApp
WhatsApp использует E2E-шифрование, но сервер принадлежит Meta. Метаданные собираются для рекламного профилирования. Бэкапы в облаке по умолчанию не зашифрованы. Требует номер телефона. На self-hosted Matrix метаданные остаются на твоём сервере.

### По сравнению с Signal
Signal — эталон шифрования, но сервер один и принадлежит фонду Signal Foundation. При блокировке в стране — зависимость от их прокси. Требует номер телефона. Self-hosted Matrix — ты контролируешь сервер и маршрутизацию.

### Что даёт self-hosted Matrix
- Полный контроль над данными и инфраструктурой
- Никакой привязки к телефону — полная анонимность
- Обход блокировок через relay-серверы
- Выбор клиентов — не привязан к одному приложению
- Открытый исходный код сервера и клиентов
- Возможность уничтожить все данные, удалив VPS

---

## Заметки

- **Шифрование**: в зашифрованных комнатах шифруется всё — текст, фото, видео, аудио, файлы. Сервер хранит только зашифрованные данные.
- **Security Key**: без него при потере всех сессий зашифрованная переписка будет утрачена навсегда.
- **Федерация**: отключена (`federation_domain_whitelist: []`). Для включения — убрать эту строку и перезапустить Synapse.
- **Server name**: нельзя изменить после начала использования. Вшивается в ID пользователей, комнат и событий.
- **Обновление Element Web**: скачать новую версию, распаковать в `/var/www/element`, скопировать `config.json`.
- **Переключение на прямое подключение**: вернуть A-записи `matrix.example.com` и `element.example.com` на IP основного сервера, relay можно выключить.

---

## Полезные ссылки

- [Matrix.org](https://matrix.org/) — официальный сайт протокола Matrix
- [Synapse](https://github.com/element-hq/synapse) — исходный код сервера
- [Element Web](https://github.com/element-hq/element-web) — исходный код веб-клиента
- [Каталог клиентов](https://matrix.org/ecosystem/clients/) — все известные клиенты Matrix
- [Synapse Admin](https://admin.etke.cc) — веб-панель администрирования
- [Документация Synapse](https://element-hq.github.io/synapse/latest/) — полная документация сервера
