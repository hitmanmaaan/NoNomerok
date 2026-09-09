# Горздрав Telegram Bot

Telegram-бот для записи к врачу через систему Горздрав (Санкт-Петербург).

## Возможности

- 👤 **Регистрация** — ФИО, дата рождения, СНИЛС (хранятся локально)
- 📋 **Запись к врачу** — выбор района → поликлиника → специальность → врач → время → подтверждение
- 🔔 **Мониторинг талонов** — фоновый воркер проверяет появление талонов в нужное время и присылает уведомление
- 🗂 **Управление мониторингами** — просмотр и отмена активных задач

## Структура проекта

```
gorzdrav-bot/
├── cmd/bot/                 # точка входа
├── config/                  # загрузка конфига из ENV
├── internal/
│   ├── domain/              # сущности и состояния FSM
│   ├── gorzdrav/            # HTTP-клиент к API Горздрава
│   ├── storage/
│   │   ├── storage.go       # интерфейсы репозиториев
│   │   └── postgres/        # реализация на PostgreSQL + migrate
│   ├── service/             # бизнес-логика
│   │   ├── user.go
│   │   ├── appointment.go
│   │   └── monitor.go
│   ├── bot/
│   │   ├── fsm/             # хелпер для работы с состояниями
│   │   ├── keyboards/       # inline + reply клавиатуры с пагинацией
│   │   └── handlers/        # registry + register/appointment/monitor
│   └── scheduler/           # фоновый воркер мониторинга
├── migrations/              # SQL-миграции (применяются при старте)
├── Dockerfile
├── docker-compose.yml
└── .env.example
```

## Быстрый старт

### 1. Настройка переменных окружения

```bash
cp .env.example .env
# Отредактируйте .env — укажите TELEGRAM_TOKEN
```

### 2. Запуск через Docker Compose

```bash
docker compose up --build -d
```

Бот автоматически применит миграции и запустится.

### 3. Локальная разработка без Docker

```bash
# Запустите PostgreSQL локально или через Docker:
docker run -d --name pg \
  -e POSTGRES_DB=gorzdrav \
  -e POSTGRES_USER=gorzdrav \
  -e POSTGRES_PASSWORD=gorzdrav_secret \
  -p 5432:5432 postgres:16-alpine

# Установите переменные и запустите:
source .env
go run ./cmd/bot
```

## Переменные окружения

| Переменная        | Описание                                      | Пример                                      |
|-------------------|-----------------------------------------------|---------------------------------------------|
| `TELEGRAM_TOKEN`  | Токен бота от @BotFather                      | `123456:ABC-DEF...`                         |
| `DATABASE_DSN`    | PostgreSQL DSN                                | `postgres://user:pass@host:5432/db?sslmode=disable` |
| `WORKER_INTERVAL` | Базовый интервал тика воркера (секунды)       | `60`                                        |

> Каждый мониторинг имеет **собственный интервал** (минуты), выбираемый пользователем при создании.
> `WORKER_INTERVAL` — это как часто цикл _просыпается_ и проверяет, какие задачи пора опросить.

## Архитектура FSM

Состояние диалога каждого пользователя хранится в PostgreSQL (колонка `state` + `state_data` JSONB).
Это позволяет перезапускать бота без потери контекста разговора.

```
StateIdle
  ├─► StateRegisterLastName → FirstName → MiddleName → Birthdate → SNILS → Idle
  ├─► StateBookSelectDistrict → LPU → Specialty → Doctor → Time → Confirm → Idle
  └─► StateMonitorSelectDistrict → LPU → Specialty → Doctor → TimeFrom → TimeTo → Interval → Idle
```

## Заметки по безопасности

- СНИЛС и дата рождения хранятся в открытом виде в БД — при необходимости добавьте шифрование (например, `pgcrypto`).
- Данные передаются в API Горздрава только в момент записи — бот не кешируует медицинские данные сторонних сервисов.
- `esiaId` в запросе на запись передаётся как `"000000000"` — уточните у Горздрава нужен ли реальный ESIA-идентификатор.
