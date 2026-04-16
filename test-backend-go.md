# Тестовое задание, Backend стажёр

Ориентировочное время: 10-12 часов. Это не дедлайн, а честная прикидка. Если ушло больше, потому что хотелось разобраться глубже, просто напиши в сопроводительном письме. 

## На чём писать

У нас Go. Если ты Python-разработчик и хочешь попробовать, ок, пиши на FastAPI или Flask. Требования к архитектуре, безопасности и качеству остаются теми же, просто мысленно заменяй `pgx/v5` на `asyncpg`, `net/http` на `FastAPI`, `gofmt` на `ruff`. Смысл не меняется.

Go сам по себе несложный, но писать идиоматично сразу не получится. Из-за этого Go-решение нам приятнее: видно, как ты мыслишь в стиле, в котором будешь работать.

Дальше текст идёт под Go. Если берёшь Python, адаптируй без потери смысла.

## Про что задание

Мы пишем бэкенд для платформы, где студенты оставляют отзывы о преподавателях, а администрация смотрит аналитику. Те самые анкеты, которые вы в ВШЭ заполняете четыре раза в год. У нас в базе больше 178 тысяч отзывов и около 7 тысяч преподавателей, проект живёт на sopanalytics.miem.tv.

Тестовое это упрощённая копия небольшого куска нашего прода. Ни одного лишнего требования: всё, что просим, реально есть в проекте и реально пригождается.

## Задача

Сделай REST-сервис отзывов о преподавателях. Регистрация, логин с JWT, создание отзывов с оценкой и текстом, просмотр статистики. Админка для блокировки подозрительных пользователей.

### Модель данных

Четыре таблицы: `users`, `teachers`, `feedbacks`, `refresh_tokens`. Вот набросок:

```
users
  id            bigserial primary key
  email         text unique not null       -- храним в нижнем регистре
  password_hash text not null              -- bcrypt cost 10
  role          text not null default 'user'  -- 'user' или 'admin'
  is_blocked    boolean not null default false
  created_at    timestamptz not null default now()
  updated_at    timestamptz not null default now()

teachers
  id        bigserial primary key
  full_name text not null
  faculty   text not null
  email     text                           -- необязательный

feedbacks
  id          bigserial primary key
  teacher_id  bigint not null references teachers(id) on delete cascade
  user_id     bigint not null references users(id) on delete cascade
  rating      smallint not null check (rating between 1 and 5)
  comment     text not null check (length(comment) between 10 and 2000)
  created_at  timestamptz not null default now()
  unique (teacher_id, user_id)   -- один отзыв от одного юзера на одного препода

refresh_tokens
  id         bigserial primary key
  user_id    bigint not null references users(id) on delete cascade
  token_hash text not null unique  -- sha256 от самого токена, не plain
  expires_at timestamptz not null
  revoked_at timestamptz            -- null пока активен
  created_at timestamptz not null default now()

-- полезные индексы
create index feedbacks_teacher_idx on feedbacks(teacher_id);
create index feedbacks_user_idx on feedbacks(user_id);
create index refresh_tokens_user_idx on refresh_tokens(user_id) where revoked_at is null;
```

`updated_at` обновляй триггером или в коде, как удобнее.

### Эндпоинты

Все ответы в JSON. Успех это 2xx. Ошибка это 4xx с телом `{"error": "понятный текст"}`. Для валидационных ошибок добавляй `"details": {...}`, где ключи это поля формы, а значения это причины. Формат ответов дальше.

#### POST /api/auth/register

```json
// запрос
{"email": "ivanov@hse.ru", "password": "Secret123"}

// успех 201
{
  "user": {"id": 42, "email": "ivanov@hse.ru", "role": "user"},
  "access_token": "eyJhbGc...",
  "refresh_token": "long-random-string",
  "access_expires_in": 900,
  "refresh_expires_in": 604800
}

// ошибки
400 {"error": "validation failed", "details": {"password": "минимум 8 символов, одна буква и одна цифра"}}
409 {"error": "пользователь с таким email уже существует"}
429 {"error": "слишком много попыток"}
```

Правила пароля: длина не меньше 8, хотя бы одна буква, хотя бы одна цифра. Никаких «обязательно спецсимвол», от этого пользователи страдают, а безопасности это не даёт.

Если переменная `ADMIN_EMAIL` выставлена и регистрирующийся email совпадает с ней, выдаём ему `role=admin` сразу. Удобно, чтобы поднять первого админа без ручных SQL. Если `ADMIN_EMAIL` пустой, никто автоматически не получает админку.

#### POST /api/auth/login

```json
{"email": "ivanov@hse.ru", "password": "Secret123"}

// 200: та же структура что у register
// 401 {"error": "неверный email или пароль"}
// 403 {"error": "аккаунт заблокирован"}
// 429 {"error": "слишком много попыток"}
```

Важный момент про 401. На несуществующий email и на неверный пароль отвечай одинаковым текстом. Иначе через разные тексты ошибок можно перебирать, какие email зарегистрированы. Это называется user enumeration, и это стандартная дыра, которую легко закрыть.

#### POST /api/auth/refresh

```json
{"refresh_token": "long-random-string"}

// 200: новая пара, старый refresh_token помечаем revoked
// 401 {"error": "недействительный refresh token"}
```

Рефреш-ротация обязательна. При каждом обновлении старый refresh-токен становится невалиден, выдаётся новый. Это защита от replay: если токен утечёт, злоумышленник успеет использовать его один раз, после чего легитимный пользователь получит 401 и заметит.

#### POST /api/auth/logout

```json
{"refresh_token": "long-random-string"}
// 204 без тела
```

Отзывает конкретный refresh-токен. Access-токен мы не отзываем, он и так короткий (15 минут по умолчанию).

#### GET /api/teachers

Публичный, без авторизации. Query-параметры:

```
q        строка поиска по full_name (ILIKE '%q%', необязательно)
faculty  фильтр по факультету (точное совпадение)
limit    int от 1 до 100, по умолчанию 20
offset   int >= 0, по умолчанию 0
```

Ответ:

```json
{
  "items": [
    {
      "id": 1,
      "full_name": "Иванов Иван Иванович",
      "faculty": "факультет компьютерных наук",
      "reviews_count": 87,
      "avg_rating": 4.3
    }
  ],
  "total": 6924,
  "limit": 20,
  "offset": 0
}
```

`avg_rating` округляй до одного знака. Сортировка по умолчанию по `full_name ASC` (не по `id`, чтобы результат не зависел от порядка вставки). Если хочется, добавь параметр `sort` для `avg_rating DESC`, это бонус.

#### GET /api/teachers/{id}

Публичный. Карточка с распределением оценок.

```json
// 200
{
  "id": 1,
  "full_name": "Иванов Иван Иванович",
  "faculty": "факультет компьютерных наук",
  "email": "ivanov@hse.ru",
  "reviews_count": 87,
  "avg_rating": 4.3,
  "rating_distribution": {"1": 2, "2": 3, "3": 10, "4": 25, "5": 47}
}
// 404 {"error": "преподаватель не найден"}
```

#### POST /api/feedbacks

Защищённый, требует `Authorization: Bearer <access_token>`.

```json
// запрос
{"teacher_id": 1, "rating": 5, "comment": "Отличные лекции, всё по делу"}

// 201
{
  "id": 555,
  "teacher_id": 1,
  "rating": 5,
  "comment": "Отличные лекции, всё по делу",
  "created_at": "2026-04-16T10:30:00Z"
}

// ошибки
400 валидация
401 нет токена или просрочен
409 {"error": "вы уже оставляли отзыв на этого преподавателя"}
```

#### GET /api/feedbacks/me

Защищённый. Свои отзывы.

```json
// 200
{
  "items": [
    {"id": 555, "teacher_id": 1, "teacher_name": "Иванов И. И.", "rating": 5, "comment": "...", "created_at": "..."}
  ]
}
```

`teacher_name` подтягиваешь JOIN-ом из `teachers` по `teacher_id`. В самой таблице `feedbacks` такого поля нет, это денормализованная выдача для клиента.

#### GET /api/admin/feedbacks

Только для `role=admin`. Query-параметры для пагинации (`limit`, `offset`) и для фильтров `user_id` или `teacher_id`.

```json
// 200: та же структура что у /feedbacks/me, плюс в каждом айтеме user_email
// 403 {"error": "недостаточно прав"}
```

#### POST /api/admin/users/{id}/block

Только админ. Ставит `is_blocked=true` и отзывает все активные refresh-токены пользователя одной транзакцией.

```json
// 204 без тела
// 404 {"error": "пользователь не найден"}
```

Про access-токен. В прошлый раз выданный юзеру access живёт ещё максимум 15 минут. Это сознательный компромисс: мы не ведём blocklist выданных access-токенов, потому что это усложняет архитектуру ради 15 минут уязвимости. Если хочешь закрыть этот зазор, есть два варианта. Первый, проверять `is_blocked` из БД в auth-middleware на каждый запрос (дополнительный SELECT везде). Второй, держать JTI-blocklist в Redis. Для тестового мы считаем компромисс приемлемым, но если реализуешь один из вариантов, напиши в README почему.

### Middleware

Минимум три штуки.

**Логирование**. На каждый запрос одна строка в stdout, вида:

```
2026-04-16T10:30:00Z method=POST path=/api/auth/login status=401 duration=23ms remote=192.168.1.1
```

Не городи structured logging через zap или zerolog для тестового. `log.Printf` отлично справится.

**CORS**. Разрешаем origin из переменной `CORS_ALLOWED_ORIGIN` (дефолт `*` для деведения). Методы `GET, POST, OPTIONS`. Заголовки `Content-Type, Authorization`. На preflight `OPTIONS` отвечаем сразу 204, без заголовка `Access-Control-Allow-Credentials`, потому что кукисы нам не нужны.

**Auth**. Проверяет `Authorization: Bearer <token>`, парсит JWT, кладёт `userID` и `role` в контекст запроса через `context.WithValue`. Если токена нет или он невалидный, возвращает 401. Применяй только к защищённым роутам, а не ко всему подряд.

**Rate limiting**. На `/api/auth/register` и `/api/auth/login` не больше 5 попыток в минуту с одного IP. IP бери из `X-Real-IP` если заголовок есть (мы планируем за nginx), иначе из `r.RemoteAddr`. Осторожно: `r.RemoteAddr` содержит порт, его надо отрезать через `net.SplitHostPort`, иначе каждое новое TCP-соединение с клиента будет считаться новым IP (порт-то меняется, а IP нет). Счётчики в памяти через `sync.Map` или самописную структуру, Redis для тестового излишен.

### Переменные окружения

Обязательные:

```
DATABASE_URL=postgres://user:pass@localhost:5432/dbname?sslmode=disable
JWT_SECRET=<минимум 32 символа, проверяй на старте>
```

Опциональные с дефолтами:

```
PORT=8080
ADMIN_EMAIL=
CORS_ALLOWED_ORIGIN=*
ACCESS_TOKEN_DURATION=15m
REFRESH_TOKEN_DURATION=168h
BCRYPT_COST=10
RATE_LIMIT_ATTEMPTS=5
RATE_LIMIT_WINDOW=1m
```

Все переменные читаются на старте. Если `JWT_SECRET` короче 32 символов или `DATABASE_URL` пустой, сервис падает с понятной ошибкой. Лучше упасть громко, чем работать странно потом.

### Структура проекта

Примерно так:

```
.
├── cmd/server/main.go          # точка входа, сборка зависимостей, http.Server
├── internal/auth/              # JWT, bcrypt, middleware
├── internal/handler/           # HTTP-хэндлеры
├── internal/model/             # структуры User, Teacher, Feedback
├── internal/store/             # работа с БД через pgx
├── internal/config/            # чтение env и валидация
├── migrations/
│   ├── 001_init.sql
│   └── ...
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── .gitignore
├── Makefile                    # необязательно: make run, make test, make migrate
├── go.mod
├── go.sum
└── README.md
```

Мне интересно посмотреть, как ты делишь ответственность. Обратная крайность тоже плохая: не городи 15 слоёв абстракций для обычного CRUD.

### Миграции

Отдельные SQL-файлы в `migrations/`, пронумерованы по порядку. Раннер либо свой на 30 строк, либо через `github.com/golang-migrate/migrate`, либо через встроенный `embed.FS`. На старте приложения миграции применяются автоматически, это удобно в проде.

Откат миграций для тестового не просим, но если сделаешь файлы `_down.sql`, будет плюс.

### Docker

`docker-compose.yml` должен поднимать Postgres и приложение одной командой `docker compose up`. После этого сервис отвечает на `http://localhost:8080`. У Postgres обязателен healthcheck с `pg_isready`. Приложение ждёт готовности БД через `depends_on: condition: service_healthy`.

`Dockerfile` многоэтапный: builder с Go SDK собирает бинарник, runtime это `alpine` или `distroless` с одним бинарником внутри. Для Go цель размера образа это 50 МБ, реально получается 20-25. Если берёшь Python, честный размер около 150 МБ, отдельно цель по размеру не держим.

## Бонусы

Ничего не обязательно. Чем больше, тем лучше.

**Swagger или OpenAPI**. Файл `openapi.yaml` с описанием всех эндпоинтов и примерами. У нас swagger есть в проде, экономит часы объяснений смежникам.

**Тесты**. Table-driven тесты на хэндлеры авторизации и на валидацию. Интеграционные тесты через `testcontainers-go` поднимают настоящий Postgres в Docker для теста, это уровень middle. Для стажёра хватит пары юнитов.

**Graceful shutdown**. На `SIGTERM` допринимаем активные запросы 10 секунд через `http.Server.Shutdown`, потом закрываемся.

**Prepared statements** для часто используемых запросов. Не магический кэш, а явные `conn.Prepare` там, где это реально нужно.

**Request ID**. На каждый входящий запрос генерируешь UUID, кладёшь в контекст, выводишь в лог. Если клиент прислал `X-Request-ID`, используем его (но валидируй длину и набор символов, иначе потенциальный log injection). Помогает потом дебажить по логам.

**Метрики**. Эндпоинт `/metrics` с базовыми prometheus-метриками: `http_requests_total`, `http_request_duration_seconds`. Библиотека `github.com/prometheus/client_golang`.

**Audit log**. Пятая таблица, куда пишешь важные действия: логин, блокировка, revoke токенов. Поля `actor_id, action, target_id, metadata jsonb, created_at`. Админский эндпоинт `GET /api/admin/audit-log` для просмотра.

## Что сдавать

Репозиторий на GitHub или GitLab. История коммитов осмысленная, без одного коммита `initial` с 40 файлами.

В корне `README.md` с разделами:

1. Что это в двух предложениях.
2. Что нужно на машине (Go 1.23+, Docker и тд).
3. Как поднять: команды по шагам, от `git clone` до первого успешного `curl`.
4. Примеры запросов. Несколько `curl` с ожидаемыми ответами, чтобы проверяющий мог потыкать.
5. Описание структуры кода: почему такие пакеты, где что лежит.
6. Спорные решения. Если где-то выбирал одно из двух и сомневался, объясни почему.
7. Что бы сделал дальше за ещё один день.

`.env.example` с перечнем переменных и комментариями. Настоящий `.env` не коммить.

`.gitignore` в порядке: никаких `*.db`, `.env`, бинарников в репозитории.

## Частые затыки

**Сервис падает на старте с `JWT_SECRET must be at least 32 characters`.** Сгенерируй нормальный секрет: `openssl rand -hex 32` даст ровно 64 символа. Положи в `.env`.

**Rate limit режет даже первый запрос.** Типичная причина: IP из `r.RemoteAddr` берётся с портом (`127.0.0.1:54321`), а порт на каждый новый curl разный. Каждый запрос видится как новый IP до какого-то лимита. Оторви порт через `net.SplitHostPort(r.RemoteAddr)`, возьми `host`.

**Postgres в Docker не поднимается.** `docker compose logs postgres`. Обычно это либо уже занятый порт 5432 на хосте (поменяй на 5433), либо несовпадение паролей между `POSTGRES_PASSWORD` и `DATABASE_URL`.

**409 на регистрацию, которую ты вроде не делал.** Email уже есть в базе из прошлой попытки. Либо `docker compose down -v` (стирает volume), либо `DELETE FROM users WHERE email = 'xxx'` руками.

## Критерии приёмки

Мы запускаем тестовое ровно так:

```
git clone <твой репо>
cd <папка>
cp .env.example .env
docker compose up -d
```

Через минуту `http://localhost:8080/api/teachers` должен отдать JSON со списком (пустой список тоже ок, главное не 500). Если по этой инструкции не поднялось, мы идём читать README. Если и по README не поднимается, это серьёзный минус.

Дальше смотрим код.

**Безопасность.** Параметризованные запросы везде. Пароли через bcrypt. JWT с проверкой expiration и signing method. Rate limiting реальный, а не заглушка. Никакого `fmt.Sprintf` вокруг SQL, ни разу.

**Обработка ошибок.** Ошибка от БД оборачивается через `%w`, логируется и превращается в 500 с общим текстом для клиента, без утечки деталей наружу. Юзер-фейсинг ошибки конкретные и полезные.

**Идиоматичность.** Код проходит `gofmt` и `go vet`. Имена переменных короткие в маленьких скоупах, длиннее в больших. Интерфейсы определяются на стороне потребителя. Функции возвращают структуры, принимают интерфейсы.

**Документация.** README реально отвечает на вопросы, которые возникают при первом знакомстве с проектом.

**Тесты.** Если есть, запускаются через `go test ./...` без хаков с окружением. Если нет, в README честно написано почему.

## Про AI

LLM использовать можно и нужно, это нормальный инструмент. Но пиши свой код, а не копируй куски, которые не понимаешь. На интервью мы будем разбирать твоё тестовое и задавать вопросы вроде «почему ты тут `sync.Mutex`, а не `sync.RWMutex`», «почему `context.Background()` в горутине». Быстро видно, где человек, а где склеенные ответы.

«Подсмотрел у ChatGPT, сам бы не додумался» это хороший ответ. Попытка выдать чужое за своё это плохой ответ. Мы все подсматриваем, это нормальная практика.

## Куда присылать

Telegram: @S0lar1ne.

В сообщении: что получилось хорошо, что вызвало трудности, сколько реально ушло, что бы сделал иначе, ссылку на ваш репозиторий.

Пинайте в личку, возможно сообщение потерялось.
