# LogMaster

Лёгкий self-hosted сервис для анализа архивных логов Linux: загрузка `.tar.gz` / `.zip` / `.gz` / `.log`,
фоновая распаковка и потоковый парсинг, поиск по уровню, дате, тексту и Regex.
Доступ через веб-интерфейс, REST API и Telegram-бота.

## Быстрый старт (локально, без Docker и Redis)

```bash
pip install -r requirements.txt
LM_EAGER=1 uvicorn app.main:app --reload      # SQLite, обработка в фоне FastAPI
# веб-интерфейс: http://localhost:8000   документация API: http://localhost:8000/docs
pytest                                         # тесты
```

## Продакшен (Docker Compose)

```bash
cp .env.example .env        # задайте SECRET_KEY, POSTGRES_PASSWORD, BOT_TOKEN
docker compose up -d --build
```
Поднимаются: nginx, api (FastAPI), worker (Celery), beat (очистка по сроку хранения), bot (aiogram), postgres, redis.

## Структура

| Путь | Назначение |
|---|---|
| `app/main.py` | REST API (FastAPI) |
| `app/services/archive.py` | безопасная распаковка (Zip Slip, архивные бомбы, симлинки) |
| `app/services/parser.py` | потоковый парсер: Python/Java-логи, nginx access/error, syslog, многострочные трассировки |
| `app/services/pipeline.py` | распаковка → парсинг → пакетная запись в БД (по 5000) |
| `app/tasks.py` | Celery: обработка файла и периодическая очистка |
| `bot/main.py` | Telegram-бот |
| `web/index.html` | SPA на Vue 3 + Tailwind (CDN, без сборки) |
| `tests/` | pytest: парсер, архивы, API |

## API

| Метод | Адрес | Назначение |
|---|---|---|
| POST | `/api/v1/auth/register`, `/auth/login` | регистрация, вход (JWT) |
| POST | `/api/v1/auth/telegram` | вход через Telegram Login Widget (проверка подписи) |
| POST | `/api/v1/logs/upload` | загрузка архива |
| GET | `/api/v1/logs` | список своих файлов |
| GET | `/api/v1/logs/{id}/status` | статус: processing / done / failed |
| GET | `/api/v1/logs/search` | `file_id, query, regex, level, date_from, date_to, cursor, limit` |
| GET | `/api/v1/logs/{id}/parsed` | все записи файла (постранично) |
| GET | `/api/v1/logs/{id}/stats` | счётчики по уровням, частые ошибки |
| DELETE | `/api/v1/logs/{id}` | удалить файл |

Загрузка прямо с сервера (например, из cron после ротации):

```bash
TOKEN=$(curl -s localhost/api/v1/auth/login -H 'Content-Type: application/json' \
        -d '{"username":"me","password":"secret123"}' | jq -r .access_token)
curl -H "Authorization: Bearer $TOKEN" -F file=@/var/log/app.log.1.gz localhost/api/v1/logs/upload
```

## Telegram-бот

Отправьте архив документом (до 20 МБ — ограничение облачного Bot API), затем `/status`, `/errors`,
`/search слово`, `/report`. Для крупных файлов используйте веб или API. `ALLOWED_TELEGRAM_IDS` ограничивает круг пользователей.

## Безопасность

- Проверка путей, ссылок, числа файлов и реального распакованного объёма (не доверяем заголовкам архива).
- Лимит размера загрузки (`MAX_UPLOAD_MB`, nginx `client_max_body_size`).
- Распакованные файлы удаляются сразу после парсинга; исходники — по `RETENTION_DAYS`.
- Каждый пользователь видит только свои файлы; пароли — scrypt; JWT с ограниченным сроком.
- Задайте длинный `SECRET_KEY` (≥ 32 символов) и работайте за HTTPS.

## Известные ограничения

- Regex в режиме SQLite фильтруется в Python порциями; на PostgreSQL для больших объёмов стоит перейти на оператор `~`.
- Нет RBAC, графиков и AI-анализа (вне MVP).
