# Template: databases/backend_db.py

One file per database connection. Name the file `<database_name>_db.py`.
Prefix the engine, session, and dependency with the same `<database_name>_` prefix.
`Base` lives here — all models import it from this file.

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session, DeclarativeBase

from app.config.settings import settings


class Base(DeclarativeBase):
    pass


backend_engine = create_engine(
    settings.DATABASE_URL,
    pool_size=settings.DB_POOL_SIZE,
    max_overflow=settings.DB_MAX_OVERFLOW,
    pool_timeout=settings.DB_POOL_TIMEOUT,
    pool_recycle=settings.DB_POOL_RECYCLE,
    pool_pre_ping=settings.DB_POOL_PRE_PING,
    echo=settings.DB_ECHO,
)

BackendSessionLocal = sessionmaker(
    bind=backend_engine,
    autocommit=False,
    autoflush=False,
)


def get_backend_db():
    db = BackendSessionLocal()
    try:
        yield db
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()
```

## Rules

- Never use `AsyncSession` or `async_sessionmaker`
- Never call `db.commit()` inside helpers — commit in the router after all helper calls succeed
- One `get_<database_name>_db` dependency per database connection file
- Import `Session` from `sqlalchemy.orm`, not from SQLAlchemy Core
