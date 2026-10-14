# aiogram-bot-template

A production-minded template for Telegram bots: **aiogram 3**, **async SQLAlchemy 2**,
**Alembic**, **uv**. It runs out of the box (long polling + SQLite) and scales up to
webhook + PostgreSQL + Redis without code changes, only configuration.

## Features

- **Layered structure**: handlers → repositories → models. No SQL in handlers.
- **Async SQLAlchemy 2.0** with typed models, a constraint naming convention and a
  conflict-safe user upsert (works on PostgreSQL and SQLite).
- **Alembic** migrations with an async `env.py` and a ready initial migration.
- **Long polling by default, webhook mode** (aiohttp) when `WEBHOOK_BASE_URL` is set,
  including a `/health` endpoint.
- **FSM storage**: in memory, or Redis when `REDIS_URL` is set.
- **Middlewares**: per-user throttling, a database session per update (commit on success,
  rollback on error), automatic user registration (`user: User` is injectable in handlers).
- **Examples** you can keep or delete: commands, inline keyboards with `CallbackData`, a
  multi-step FSM form, an admin-only command, a global error handler, blocked-bot tracking.
- **Tooling**: `pydantic-settings`, ruff (lint + format), mypy (strict), pytest + pytest-asyncio,
  pre-commit, GitHub Actions CI.
- **Docker**: multi-stage image (uv, non-root user) and a compose file with PostgreSQL and Redis.

## Quick start

Requirements: [uv](https://docs.astral.sh/uv/) (it installs Python 3.12 for you).

```bash
cp .env.example .env        # put the token from @BotFather into BOT_TOKEN
uv sync                     # creates .venv and uv.lock
make run                    # applies migrations and starts the bot
```

Open your bot in Telegram and send `/start`.

Without `make`: `uv run alembic upgrade head && uv run python -m bot`.

> Commit `uv.lock`: the Docker build and CI use `uv sync --locked`.

## Docker (PostgreSQL + Redis)

```bash
uv sync                     # once: creates uv.lock, which the image build needs
cp .env.example .env        # set BOT_TOKEN and POSTGRES_PASSWORD
make up                     # docker compose up --build -d
make logs
```

The compose file overrides `DATABASE_URL` and `REDIS_URL` so the bot talks to its
neighbours, and applies migrations before every start. For a webhook, see below.

## Project structure

```
.
├── src/bot/
│   ├── __main__.py         # python -m bot
│   ├── main.py             # polling / webhook runners
│   ├── factory.py          # builds Bot, Dispatcher, FSM storage; registers middlewares
│   ├── lifecycle.py        # startup / shutdown hooks (DB check, commands, webhook)
│   ├── config.py           # settings (pydantic-settings)
│   ├── logger.py
│   ├── db/
│   │   ├── base.py         # Base, naming convention, TimestampMixin
│   │   ├── engine.py       # engine and session factory
│   │   ├── models/         # ORM models
│   │   └── repositories/   # data access
│   ├── handlers/           # routers: common, profile, form (FSM), admin, membership,
│   │                       #          errors, fallback
│   ├── middlewares/        # throttling, DB session, user registration
│   ├── filters/            # IsAdmin
│   ├── keyboards/          # inline keyboards and CallbackData
│   └── states/             # FSM state groups
├── alembic/                # async env.py and migrations
├── tests/
├── Dockerfile
├── docker-compose.yml
├── Makefile                # `make help` lists all targets
└── pyproject.toml
```

## How an update is processed

```
Telegram ─► ThrottlingMiddleware ─► DbSessionMiddleware ─► UserMiddleware ─► router tree
            (drops message floods)   (opens `session`,       (upserts user,    admin → common → profile
                                      commits at the end)     injects `user`)  → form → membership
                                                                               → errors → fallback
```

Handlers declare what they need by parameter name: `message`, `state`, `session`,
`user`, `settings`, `engine`, …

## Configuration

All settings come from environment variables or `.env` (see `.env.example`).

| Variable | Default | Description |
| --- | --- | --- |
| `BOT_TOKEN` | – (required) | Token from @BotFather |
| `ADMIN_IDS` | empty | Comma-separated Telegram IDs allowed to use admin commands |
| `DATABASE_URL` | `sqlite+aiosqlite:///./data/bot.db` | SQLAlchemy async URL. `postgres://` and `postgresql://` are rewritten to `postgresql+asyncpg://` |
| `DATABASE_ECHO` | `false` | Log every SQL statement |
| `REDIS_URL` | empty | Store FSM state in Redis (otherwise in memory) |
| `WEBHOOK_BASE_URL` | empty | Public HTTPS URL; setting it switches the bot to webhook mode |
| `WEBHOOK_PATH` | `/webhook` | Path Telegram posts updates to |
| `WEBHOOK_SECRET` | empty | Required in webhook mode; Telegram sends it in a header and the bot verifies it |
| `WEB_HOST` / `WEB_PORT` | `0.0.0.0` / `8080` | Where the webhook server listens |
| `DROP_PENDING_UPDATES` | `false` | Discard updates received while the bot was offline |
| `THROTTLE_RATE_LIMIT` | `0.5` | Seconds between messages from one user; `0` disables |
| `LOG_LEVEL` | `INFO` | `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL` |

SQLite is meant for development and tests. Use PostgreSQL in production.

## Webhook mode

1. Run the bot behind a reverse proxy with HTTPS (Telegram requires it).
2. Set `WEBHOOK_BASE_URL=https://bot.example.com` and `WEBHOOK_SECRET` (generate one with
   `python -c "import secrets; print(secrets.token_urlsafe(32))"`).
3. Publish port `8080` (uncomment `ports` in `docker-compose.yml`) and proxy the webhook path to it.

On startup the bot registers the webhook itself. Remove `WEBHOOK_BASE_URL` to go back to
long polling; the webhook is deleted automatically.

## Database and migrations

```bash
make migrate                         # apply migrations
make revision m="add orders table"   # autogenerate a migration after changing models
```

Migrations only need `DATABASE_URL`, not `BOT_TOKEN`. Always review the generated file. New
revisions are formatted with ruff automatically (`[post_write_hooks]` in `alembic.ini`).

## Adding a feature

1. **Model**: add it to `src/bot/db/models/` and import it in `db/models/__init__.py`.
2. **Migration**: `make revision m="..."`, review it, `make migrate`.
3. **Repository**: put queries into `src/bot/db/repositories/`; repositories do not commit,
   the middleware does.
4. **Handlers**: create a module in `src/bot/handlers/` with a `router`, include it in
   `handlers/__init__.py` **before** `fallback.router`.

Do not call `session.commit()` in handlers unless you really need an intermediate commit:
`DbSessionMiddleware` commits when the handler returns and rolls back when it raises.

## Development

```bash
make hooks       # install pre-commit hooks
make fmt         # format and auto-fix
make check       # lint + mypy + tests (what CI runs)
```

## Notes

- Python 3.12+. Dependencies are declared in `pyproject.toml`; `uv sync` resolves them.
- The throttling state is in process memory. With several bot instances use a shared store.
- Bot messages in the examples are in Russian; texts live next to the handlers, so they are
  easy to replace or move into an i18n layer.
