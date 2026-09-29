# Template: Alembic migrations

Alembic manages database schema migrations. One `migrations/` folder per service.

## Folder structure

```
<service-root>/
  migrations/
    versions/       # auto-generated migration scripts
    env.py          # wired to app models and settings
    script.py.mako  # migration script template (auto-generated)
  alembic.ini       # Alembic config file (auto-generated)
```

## Setup

```bash
alembic init migrations
```

## env.py — wire to app models and settings

`alembic init` generates a standard `env.py`. Make these three targeted changes to it:

**1. Add imports at the top (after existing imports):**
```python
from app.config.settings import settings
from app.databases.backend_db import Base
import app.models  # noqa: F401 — side-effect import registers models on Base.metadata
```

**2. Set the database URL from settings (replaces the alembic.ini URL):**
```python
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)
```

**3. Set the target metadata (replaces `target_metadata = None`):**
```python
target_metadata = Base.metadata
```

Everything else in the generated `env.py` stays as-is.

## Rules

- Always review the auto-generated migration script before applying — autogenerate misses some changes (e.g. check constraints, custom types)
- Never edit a migration that has already been applied to production — create a new one
- `DATABASE_URL` is read from `settings` — never hard-code it in `alembic.ini` or `env.py`
- `import app.models` in `env.py` triggers model registration on `Base.metadata` — for this to work, `app/models/__init__.py` must import every model class (e.g. `from app.models.project import Project`); a missing model will not be detected by autogenerate

## Shared database across multiple services

When multiple services share one database, only one service owns and runs migrations — all others connect and query but never call `alembic upgrade head`.

- **Owner service**: holds all models (its own + shared), runs `alembic upgrade head` on deploy, is always deployed first
- **Other services**: reflect shared tables at startup using SQLAlchemy metadata reflection, have no `migrations/` folder, never run Alembic
- Never let two services run `alembic upgrade head` against the same database — this causes race conditions and corrupts the Alembic version table

### Non-owner service — reflecting shared tables

Non-owner services use `metadata.reflect()` to pull the schema from the database at startup. Add `DB_SCHEMA` to `settings`:

```python
# config/settings.py
DB_SCHEMA: str
```

Reflect shared tables in the database file:

```python
# databases/backend_db.py
from sqlalchemy import MetaData
from app.databases.backend_db import backend_engine
from app.config.settings import settings

shared_metadata = MetaData()
shared_metadata.reflect(bind=backend_engine, schema=settings.DB_SCHEMA, views=True)
```

Access reflected tables by name:

```python
users_table = shared_metadata.tables[f"{settings.DB_SCHEMA}.users"]
```

Use `shared_metadata.tables` to query — no model class needed:

```python
from sqlalchemy.orm import Session

def get_user_by_id(db: Session, user_id: int):
    users_table = shared_metadata.tables[f"{settings.DB_SCHEMA}.users"]
    return db.execute(users_table.select().where(users_table.c.id == user_id)).first()
```

- `views=True` includes views in the reflection — useful for read-only projections across service boundaries
- Reflection runs at startup — the database must be reachable when the service boots
- Never run `alembic upgrade head` in a non-owner service
