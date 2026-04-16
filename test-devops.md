# Тестовое задание, DevOps стажёр

Ориентировочное время: 4-6 часов. Если дольше, потому что видишь Docker Compose или nginx впервые, это нормально. Честно напиши в сопроводительном письме реальные часы.

Задание самодостаточное. Код писать не надо, мы даём готовое мини-приложение на 40 строк Go.

## Про что задание

У нас в проде простой сетап на одной Ubuntu-машине: веб-приложение, PostgreSQL, nginx как reverse proxy с SSL. Никакого Kubernetes, облачных балансировщиков, service mesh. Нам подходит, и мы не планируем менять.

Тестовое это тренировка именно такого подхода. Ты собираешь production-стенд целиком через Docker Compose: три контейнера (app, postgres, nginx), SSL, firewall, бэкапы, README. Сервер должен пережить `sudo reboot` без твоего вмешательства.

Задание сознательно срезано под 4-6 часов. Мы не гоняем стажёров по systemd-юнитам и hardening-директивам. Если после тестового захочешь копать глубже, есть бонусы в конце.

## Где разворачивать

Нужна Ubuntu 22.04 или 24.04. Самый дешёвый путь это multipass, ставится одной командой на Mac, Linux или Windows:

```
multipass launch 24.04 --name devtest --cpus 2 --memory 2G --disk 10G
multipass shell devtest
```

Альтернативы: Hetzner CX11 за 4 евро в месяц (с публичным IP удобно проверять снаружи), DigitalOcean Basic Droplet, локальная VM в VirtualBox или UTM, Yandex Cloud Nano.

Репозиторий на GitHub или GitLab, туда складываешь все конфиги.

## Приложение

Возьми наше мини-приложение, два файла.

`main.go`:

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "net/http"
    "os"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

func main() {
    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" {
        log.Fatal("DATABASE_URL required")
    }

    ctx := context.Background()
    pool, err := pgxpool.New(ctx, dsn)
    if err != nil {
        log.Fatalf("pool: %v", err)
    }
    defer pool.Close()

    http.HandleFunc("/api/hello", func(w http.ResponseWriter, r *http.Request) {
        var now time.Time
        if err := pool.QueryRow(r.Context(), "SELECT NOW()").Scan(&now); err != nil {
            http.Error(w, "db error", 500)
            return
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]any{"message": "hello", "time": now})
    })

    http.HandleFunc("/api/health", func(w http.ResponseWriter, r *http.Request) {
        if err := pool.Ping(r.Context()); err != nil {
            http.Error(w, "not ok", 503)
            return
        }
        w.Write([]byte("ok"))
    })

    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    log.Printf("listening on :%s", port)
    log.Fatal(http.ListenAndServe(":"+port, nil))
}
```

`go.mod`:

```
module demo
go 1.23
require github.com/jackc/pgx/v5 v5.7.1
```

И `Dockerfile` к нему, многостадийный:

```dockerfile
FROM golang:1.23-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /out/app .

FROM alpine:3.20
RUN apk add --no-cache ca-certificates
COPY --from=build /out/app /app
EXPOSE 8080
ENTRYPOINT ["/app"]
```

Финальный образ получится около 20 МБ. Собери и проверь:

```bash
docker build -t demo-app .
```

Можешь взять любое своё или open-source приложение, которое слушает HTTP и ходит в Postgres. Деплой важнее кода.

## Задача

### Архитектура

```
Интернет (443 HTTPS)
    ↓
nginx (контейнер, 80→443 redirect, 443 SSL → app:8080)
    ↓
app (контейнер на внутренней сети Docker)
    ↓
postgres (контейнер на внутренней сети Docker)
```

Наружу торчат только 22 (SSH), 80 (redirect), 443 (HTTPS). Postgres и app не имеют портов, проброшенных на хост. Общаются через внутреннюю сеть Compose, туда снаружи никто не достучится.

### Что собираешь

Один `docker-compose.yml` с тремя сервисами, всё лежит в `/opt/app/` на сервере.

#### Структура на сервере

```
/opt/app/
├── docker-compose.yml
├── .env                     (пароль postgres, не в git)
├── nginx/
│   └── app.conf             конфиг reverse proxy
├── ssl/
│   ├── fullchain.pem
│   └── privkey.pem
├── app/                     если собирал наше мини-приложение
│   ├── main.go
│   ├── go.mod
│   └── Dockerfile
└── html/
    └── index.html           статика, которую отдаёт nginx
```

#### SSL

Если у сервера публичный домен, используй Let's Encrypt (в README опиши как). Для multipass/локальной VM без DNS делай самоподписанный:

```bash
cd /opt/app
sudo mkdir -p ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout ssl/privkey.pem \
    -out ssl/fullchain.pem \
    -subj "/CN=devtest.local"
sudo chmod 600 ssl/privkey.pem
```

#### docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: app-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  app:
    build: ./app
    container_name: app
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      PORT: "8080"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - backend

  nginx:
    image: nginx:alpine
    container_name: app-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/app.conf:/etc/nginx/conf.d/default.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
      - ./html:/var/www/html:ro
    depends_on:
      - app
    networks:
      - backend

networks:
  backend:

volumes:
  postgres-data:
```

Разбор по пунктам.

Сервис `postgres` не публикует порт наружу (`ports:` отсутствует). Postgres доступен только другим контейнерам через имя хоста `postgres` на внутренней сети `backend`. Снаружи его не видно, что и нужно.

Сервис `app` тоже без проброса портов. Только nginx знает, что app живёт на `http://app:8080` внутри сети. Если кто-то попытается достучаться до приложения минуя nginx, у него не получится.

Сервис `nginx` единственный публикует порты 80 и 443 на хост. Конфиг и SSL примонтированы read-only, потому что менять их из контейнера не надо.

`restart: unless-stopped` означает, что контейнер автоматически поднимается при старте докера. Если ты включишь Docker в systemd (`sudo systemctl enable docker`), после ребута вся тройка поднимется сама.

Healthcheck у Postgres настоящий через `pg_isready`. Сервис `app` ждёт `service_healthy`, то есть стартует только когда Postgres реально готов принимать подключения. Без этого app пытался бы подключиться до того, как Postgres стартанул, и упал бы с ошибкой.

Named volume `postgres-data` держит данные БД отдельно от контейнера. Пересоздашь контейнер, данные останутся.

#### .env

Рядом с `docker-compose.yml` лежит `.env` (в git не коммить):

```
POSTGRES_DB=app
POSTGRES_USER=app
POSTGRES_PASSWORD=сгенерируй_через_openssl_rand_base64_24
```

Compose автоматически подхватывает `.env` из рабочей директории, переменные подставляются в `${...}` выражения.

#### nginx/app.conf

```nginx
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name _;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;

    server_tokens off;
    client_max_body_size 10M;

    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;

    location /api/ {
        proxy_pass http://app:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_read_timeout 60s;
        proxy_connect_timeout 5s;
    }

    location / {
        root /var/www/html;
        try_files $uri $uri/ /index.html;
    }
}
```

`proxy_pass http://app:8080` работает, потому что `app` это имя сервиса в compose, внутренний DNS Docker резолвит его в IP контейнера. Никаких `localhost`, никаких хост-айпи.

Заголовки безопасности базовые. HSTS говорит браузеру приходить только по HTTPS следующие 365 дней. `X-Content-Type-Options: nosniff` отключает MIME-снифинг. `X-Frame-Options: DENY` защищает от clickjacking.

#### html/index.html

Простая заглушка:

```html
<!doctype html>
<html>
<head><title>Demo stand</title></head>
<body>
  <h1>Demo production stand</h1>
  <p>API at <a href="/api/hello">/api/hello</a></p>
</body>
</html>
```

#### Запуск

```bash
cd /opt/app
sudo docker compose up -d
sudo docker compose ps
```

Должны быть три контейнера в статусе `Up` (postgres дополнительно `(healthy)`).

### Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

Статус должен показать ровно три allow-правила.

### SSH

В `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

Если делаешь на VPS, сначала добавь свой ключ через `ssh-copy-id user@server`, только потом выключай пароли, иначе заблокируешь себя от своего же сервера.

```bash
sudo systemctl restart ssh
```

### Бэкапы

Скрипт `/usr/local/bin/backup-db.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

source /opt/app/.env

BACKUP_DIR=/var/backups/app
KEEP_DAYS=7
DATE=$(date +%Y%m%d-%H%M%S)
FILE="$BACKUP_DIR/db-$DATE.sql.gz"

mkdir -p "$BACKUP_DIR"
docker exec app-postgres pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" | gzip > "$FILE"

if [ ! -s "$FILE" ]; then
    echo "backup empty, abort" >&2
    exit 1
fi

find "$BACKUP_DIR" -type f -name 'db-*.sql.gz' -mtime +$KEEP_DAYS -delete
echo "backup ok: $FILE"
```

`set -euo pipefail` это стандартная защита: упасть при первой ошибке (`-e`), поймать ошибки в pipe (`-o pipefail`), упасть при попытке использовать неопределённую переменную (`-u`). Без этого скрипт может тихо продолжить работать после pg_dump-ошибки, и ты получишь пустой файл бэкапа.

Права и запуск по расписанию:

```bash
sudo chmod 755 /usr/local/bin/backup-db.sh

# разовый тест
sudo /usr/local/bin/backup-db.sh
ls -lh /var/backups/app/

# в cron каждые сутки в 3:00 ночи
sudo crontab -e
# добавить:
0 3 * * * /usr/local/bin/backup-db.sh >> /var/log/backup-db.log 2>&1
```

Проверь восстановление: сделай дамп, удали пару строк из базы через `docker exec -it app-postgres psql ...`, накати дамп обратно (`gunzip -c dump.sql.gz | docker exec -i app-postgres psql ...`), убедись что вернулись. Не сдавать, но сам пройди цикл, на интервью спросим.

### Проверка после ребута

```bash
sudo reboot
# подожди минуту, залогинься обратно
curl -k https://localhost/api/hello
# ожидаем JSON с полями message и time
```

Если что-то не поднялось, скорее всего `systemctl enable docker` не сделан. Это главный критерий приёмки: стенд переживает ребут без ручного вмешательства.

## Частые затыки

**`docker compose up` падает с `permission denied` на `ssl/privkey.pem`.** nginx внутри контейнера запускается от пользователя `nginx`, который не может прочитать файл с правами 600. Либо `chmod 640` и поменять группу, либо положить ключ в Docker secret. Для тестового хватит `chmod 644` на privkey с комментарием в README, что так делать в проде не надо.

**`/api/hello` возвращает 502.** Nginx работает, app нет. `sudo docker compose logs app`, обычно там `DATABASE_URL` invalid или Postgres ещё не поднялся. Проверь `service_healthy` в `depends_on`, проверь что пароли в `.env` совпадают с `DATABASE_URL`.

**После ребута приложение не поднялось.** Забыл `sudo systemctl enable docker`. С `restart: unless-stopped` контейнеры поднимаются только если сам Docker запустился.

**Backup возвращает пустой файл.** `source /opt/app/.env` не прочитался или `POSTGRES_USER` пустой. Добавь проверку в начало скрипта, скажем `: "${POSTGRES_USER:?need POSTGRES_USER}"`.

**`docker compose` говорит `unknown command`.** Старый Docker без плагина compose. Поставь `curl -fsSL https://get.docker.com | sudo sh`, это ставит новый Docker с compose-плагином в одну команду.

## Что сдавать

Репозиторий:

```
.
├── README.md
├── docker-compose.yml
├── .env.example
├── nginx/
│   └── app.conf
├── html/
│   └── index.html
├── app/
│   ├── main.go
│   ├── go.mod
│   └── Dockerfile
└── scripts/
    └── backup-db.sh
```

В `README.md`:

1. Что за стенд, схема архитектуры.
2. Требования (Ubuntu 22.04 или 24.04, 1 CPU, 1 GB RAM, 10 GB диск).
3. Пошаговая установка от `ssh user@host` до `curl -k https://host/api/hello`, копипастой проверяемо.
4. Как смотреть логи, как сделать бэкап вручную, как восстановить из бэкапа.
5. Troubleshooting на 502, пустой бэкап, не поднялось после ребута.
6. Обоснование двух-трёх решений. Почему не проброшен порт postgres, почему `restart: unless-stopped` а не `always`, почему самоподписанный SSL а не Let's Encrypt.
7. Что бы добавил за ещё несколько часов.

Доступ для проверки, один из двух вариантов:

- Живой сервер с публичным IP плюс публичный SSH-ключ, мы зайдём и проверим.
- Screencast 5-10 минут: чистая VM, деплой по README, работающее приложение, `sudo reboot`, приложение снова работает, ручной бэкап.

## Критерии приёмки

**Идемпотентность.** Повторный запуск `docker compose up -d` не ломает стенд, только перезапускает нужное. Если надо `rm -rf /opt/app` перед повтором, это баг.

**Безопасность.** Паролей в git нет. SSH по ключу, root выключен. Postgres и app не пробрасывают порты наружу, только nginx публикует 80/443. Privkey не `644` (или есть комментарий почему в тестовом так).

**Переживает ребут.** `sudo reboot`, ждём минуту, `curl` работает. Без этого тестовое не принимается.

**README работает.** Берём чистую VM, идём по инструкции. Если на шаге X ступор и в README нет ответа, это минус.

**Бэкап не заглушка.** `docker exec ... pg_dump ...` реально дампит базу, файл не пустой, ротация удаляет старые.

## Бонусы

Всё необязательно, чем больше, тем лучше.

**Let's Encrypt с автообновлением**, если у сервера публичный домен. Certbot в отдельном контейнере, certbot-container monthly через cron или через ACME в самом nginx.

**Fail2ban** против брутфорса SSH. Правила в `/etc/fail2ban/jail.local`, не копипаста дефолта.

**Rate limiting в nginx** на `/api/`. Директива `limit_req_zone` должна лежать в `http {}` блоке (в Docker-nginx это `/etc/nginx/nginx.conf`, можно подмонтировать свой), `limit_req` используется внутри `location {}`. Обрати внимание: **`limit_req_zone` НЕ работает внутри `server {}`**, это частая ошибка.

**Приложение не в Docker, а как systemd-сервис.** Это альтернативный подход, который мы используем в нашем проде для Go-бинарника. Собираешь бинарник, кладёшь в `/opt/app/bin/app`, делаешь systemd-юнит, включаешь `enable --now`. Postgres и nginx остаются в Docker. Такой сетап чуть сложнее, но даёт меньше overhead на app. Опиши в README плюсы и минусы в сравнении с полностью контейнерным вариантом.

**Мониторинг.** Netdata одной командой внутри контейнера или на хосте, доступ через nginx с basic-auth на пути `/monitor/`. Альтернатива это prometheus плюс node-exporter в том же compose.

**CI**. GitHub Actions или GitLab CI, на каждый push проверяет конфиги: `shellcheck` на bash, валидация nginx через `docker run --rm -v .../app.conf:... nginx -t`, `docker compose config` на сам compose.

**Ansible-плейбук**, который автоматизирует весь деплой от чистой VM до работающего стенда одной командой. Это уже серьёзный уровень для стажёра, но приятно удивит.

## Про AI

LLM можно. Каждая команда должна быть тебе понятна. На интервью спросим «почему `restart: unless-stopped` а не `always`», «что делает `set -euo pipefail`», «зачем `service_healthy` в depends_on». Не знаешь, говори честно, это нормально. Выдумываешь объяснение противоречащее логике, будет неловко.

## Куда присылать

Telegram: @контакт.

В сообщении:

1. Ссылка на репо.
2. Как проверить: IP плюс публичный SSH-ключ, или ссылка на screencast.
3. Реальное время.
4. Что вышло легко, что трудно.
5. Что бы добавил ещё.

Обычно отвечаем за 3-5 рабочих дней.
