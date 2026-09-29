# Template: config/settings.py

Flat Pydantic V2 `BaseSettings` class. All configuration loaded from `.env`.
No nested config classes. No `class Config` — use `model_config` with `SettingsConfigDict`.

```python
from typing import Annotated

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

    # Service
    SERVICE_NAME: Annotated[str, Field(
        description="Name of this service — used as the logger name and OpenAPI title",
        examples=["my-service"],
    )]
    OPENAPI_VERSION: Annotated[str, Field(
        default="3.1.0",
        description="OpenAPI specification version passed to the FastAPI constructor",
        examples=["3.1.0"],
    )]

    # Database
    DATABASE_URL: Annotated[str, Field(
        description="SQLAlchemy-compatible database URL — must be present in .env",
        examples=["postgresql://user:password@localhost:5432/mydb"],
    )]
    DB_SCHEMA: Annotated[str, Field(
        description="PostgreSQL schema name used in __table_args__ on every model",
        examples=["public"],
    )]

    # Connection pool
    DB_POOL_SIZE: Annotated[int, Field(
        default=5,
        ge=1,
        description="Number of persistent connections in the pool",
        examples=[5],
    )]
    DB_MAX_OVERFLOW: Annotated[int, Field(
        default=10,
        ge=0,
        description="Extra connections allowed beyond DB_POOL_SIZE under peak load",
        examples=[10],
    )]
    DB_POOL_TIMEOUT: Annotated[int, Field(
        default=30,
        ge=1,
        description="Seconds to wait for a connection before raising an error",
        examples=[30],
    )]
    DB_POOL_RECYCLE: Annotated[int, Field(
        default=1800,
        ge=60,
        description="Seconds before a connection is recycled to avoid stale connections",
        examples=[1800],
    )]
    DB_POOL_PRE_PING: Annotated[bool, Field(
        default=True,
        description="Emit a test query before each checkout to detect dropped connections",
        examples=[True],
    )]
    DB_ECHO: Annotated[bool, Field(
        default=False,
        description="Log all SQL statements — set False in production",
        examples=[False],
    )]

    # Add additional settings below — keep flat, no nesting


settings = Settings()
```

## Rules

- One `Settings` class, one `settings` singleton — import `settings` everywhere
- Never instantiate `Settings()` outside of this file (except in `conftest.py` for tests)
- Every field must use `Annotated[type, Field(...)]` with `description`, `examples`, and constraints (`ge`, `le`) where applicable — same rule as Pydantic schemas
- Add all new env vars as typed fields with sensible defaults where applicable
- `DB_ECHO` must default to `False` — never `True` in production
- `DATABASE_URL` has no default — it must be present in `.env` or the app will fail at startup
