---
name: fastapi-scaffold
description: Scaffold a new FastAPI service with async SQLAlchemy 2.0, Alembic, Pydantic Settings, pytest, Docker multi-stage build, Makefile, and uv. Use when creating a new Python API project.
argument-hint: "<project-name> [--no-auth] [--no-docker]"
allowed-tools: Write, Bash, Read, Glob
---

# FastAPI Project Scaffold

Generate a production-ready FastAPI project following established conventions.

## Step 1 — Parse Arguments

Extract from the user's invocation:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `project-name` | (required) | Project name in kebab-case (e.g. `my-api`) |
| `--no-auth` | false | Skip JWT auth module |
| `--no-docker` | false | Skip Dockerfile and docker-compose |

Derive:
- `package_name` = project-name with hyphens replaced by underscores (e.g. `my_api`)
- `project_dir` = `./<project-name>/`

## Step 2 — Generate Project Structure

Create the following directory tree:

```
<project-name>/
├── src/<package_name>/
│   ├── __init__.py          # __version__ = "0.1.0"
│   ├── main.py              # FastAPI app, lifespan, middleware, routers
│   ├── config.py            # Pydantic Settings from env
│   ├── database.py          # Async SQLAlchemy engine + session
│   ├── models.py            # SQLAlchemy declarative Base + example model
│   ├── schemas.py           # Pydantic request/response schemas
│   └── routes/
│       ├── __init__.py
│       └── health.py        # GET /api/v1/health
├── migrations/
│   ├── env.py               # Async Alembic env
│   ├── script.py.mako       # Migration template
│   └── versions/            # (empty dir with .gitkeep)
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # Fixtures (async client, test db session)
│   └── test_health.py       # Smoke test
├── scripts/                 # Utility scripts dir
├── pyproject.toml
├── alembic.ini
├── Makefile
├── .env.example
├── .gitignore
└── CLAUDE.md                # Dev guide for AI-assisted development
```

If `--no-auth` is NOT set, also generate:
- `src/<package_name>/auth.py` — JWT token creation/verification, password hashing with argon2
- `src/<package_name>/routes/auth.py` — login, register, refresh endpoints

If `--no-docker` is NOT set, also generate:
- `Dockerfile` — multi-stage production build
- `Dockerfile.dev` — development with hot-reload
- `docker-compose.yml` — API + PostgreSQL + optional frontend placeholder

## Step 3 — File Contents

### pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "<project-name>"
version = "0.1.0"
description = ""
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.120,<1",
    "uvicorn[standard]>=0.34,<1",
    "sqlalchemy[asyncio]>=2.0,<2.1",
    "asyncpg>=0.29,<1",
    "alembic>=1.13,<2",
    "pydantic>=2.10,<3",
    "pydantic-settings>=2.7,<3",
]

[project.optional-dependencies]
dev = [
    "pytest>=8,<9",
    "pytest-asyncio>=0.24,<1",
    "pytest-cov>=6,<7",
    "httpx>=0.28,<1",
    "ruff>=0.9,<1",
    "mypy>=1.14,<2",
]

[tool.hatch.build.targets.wheel]
packages = ["src/<package_name>"]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "C4", "UP", "ARG", "SIM"]
ignore = ["E501"]

[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]
"tests/**" = ["ARG"]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"
python_files = "test_*.py"
python_functions = "test_*"
addopts = "-v --tb=short --strict-markers"

[tool.coverage.run]
source = ["src"]
omit = ["tests/*", "*/migrations/*"]

[tool.coverage.report]
precision = 2
show_missing = true
```

If `--no-auth` is NOT set, add these to `dependencies`:
```
"argon2-cffi>=23.1,<24",
"pyjwt>=2.10,<3",
"python-multipart>=0.0.20,<1",
"python-jose[cryptography]>=3.3,<4",
```

### config.py

Use `pydantic_settings.BaseSettings` with `SettingsConfigDict`:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    database_url: str = "postgresql+asyncpg://postgres:postgres@localhost/<package_name>_db"
    cors_origins: str = "http://localhost:5173,http://localhost:3000"
    environment: str = "development"

    @property
    def cors_origins_list(self) -> list[str]:
        return [o.strip() for o in self.cors_origins.split(",")]

settings = Settings()
```

If auth is included, add `jwt_secret: str`, `jwt_algorithm: str = "HS256"`, `access_token_expire_minutes: int = 15`, `refresh_token_expire_days: int = 7`.

### database.py

```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from .config import settings

engine = create_async_engine(
    settings.database_url,
    echo=settings.environment == "development",
    pool_pre_ping=True,
    pool_size=5,
    max_overflow=10,
)

async_session_maker = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autocommit=False,
    autoflush=False,
)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

### main.py

```python
from contextlib import asynccontextmanager
from collections.abc import AsyncGenerator
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from . import __version__
from .config import settings
from .database import engine
from .routes import health

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    yield
    await engine.dispose()

app = FastAPI(
    title="<project-name>",
    version=__version__,
    docs_url="/docs" if settings.environment == "development" else None,
    redoc_url="/redoc" if settings.environment == "development" else None,
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins_list,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(health.router, prefix="/api/v1", tags=["health"])
```

If auth is included, also register the auth router at `/api/v1/auth`.

### migrations/env.py

Use the async Alembic pattern:

```python
import asyncio
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context
from src.<package_name>.models import Base
from src.<package_name>.config import settings

config = context.config
config.set_main_option("sqlalchemy.url", settings.database_url)
target_metadata = Base.metadata

# Include run_migrations_offline, do_run_migrations, run_async_migrations, run_migrations_online
# following the standard async Alembic env.py pattern with compare_type=True, compare_server_default=True.
```

### Dockerfile (multi-stage)

```dockerfile
FROM python:3.12-slim AS builder
RUN pip install --no-cache-dir uv
WORKDIR /app
COPY pyproject.toml ./
RUN uv pip install --system --no-cache .

FROM python:3.12-slim
RUN useradd --create-home --shell /bin/bash appuser
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY src/ ./src/
RUN chown -R appuser:appuser /app
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/api/v1/health')"
CMD ["uvicorn", "src.<package_name>.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### docker-compose.yml

Include services:
- `postgres` — PostgreSQL 16 with volume, health check
- `api-dev` — Dockerfile.dev with hot-reload, depends_on postgres, mounts `./src` as volume
- Profile `dev` for the api-dev service

### Makefile

```makefile
.PHONY: help up down logs build-docker db-migrate lint test type-check clean install run

help:
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "  %-15s %s\n", $$1, $$2}'

install: ## Install dependencies locally with uv
	uv venv --python 3.12
	uv pip install -e ".[dev]"

run: ## Run locally with hot-reload
	uvicorn src.<package_name>.main:app --reload

lint: ## Run ruff check and format check
	ruff check src tests
	ruff format --check src tests

format: ## Auto-format code
	ruff format src tests

type-check: ## Run mypy
	mypy src/

test: ## Run tests
	pytest

coverage: ## Run tests with coverage
	pytest --cov=src --cov-report=term-missing

check: lint type-check test ## Run all checks

db-migrate: ## Run Alembic migrations
	alembic upgrade head

db-revision: ## Create new Alembic migration (usage: make db-revision msg="add users table")
	alembic revision --autogenerate -m "$(msg)"

up: ## Start Docker services
	docker-compose --profile dev up -d

down: ## Stop Docker services
	docker-compose --profile dev down

clean: ## Stop services and remove volumes
	docker-compose --profile dev down -v
```

### .env.example

```
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost/<package_name>_db
ENVIRONMENT=development
CORS_ORIGINS=http://localhost:5173,http://localhost:3000
```

If auth: add `JWT_SECRET=change-me-in-production`.

### .gitignore

Standard Python gitignore: `__pycache__/`, `.venv/`, `*.egg-info/`, `.env`, `*.pyc`, `.mypy_cache/`, `.pytest_cache/`, `.coverage`, `htmlcov/`.

### CLAUDE.md

Generate a development guide with:
- Project overview (one sentence)
- Quick start commands (`make install`, `make run`)
- Project structure tree
- Code conventions (ruff, mypy, async patterns)
- Database workflow (Alembic commands)
- Testing (`pytest`, `httpx.AsyncClient`)
- Docker usage

## Step 4 — Post-Generation

After writing all files:

1. Tell the user the project was created at `./<project-name>/`
2. Show the quick start:
   ```
   cd <project-name>
   make install
   # Start PostgreSQL (e.g., docker-compose up -d postgres)
   make db-migrate
   make run
   ```
3. Note that they should update `.env` with real values before deploying
