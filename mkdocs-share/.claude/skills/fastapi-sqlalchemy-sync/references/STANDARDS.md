# Standards: fastapi-sqlalchemy-sync

Use this file when reviewing existing FastAPI code for compliance.
All rules are in `SKILL.md` — this file adds the common violations to watch for.

## Naming — common violations

| Element | Required | Common violation |
|---|---|---|
| DB connection files | `<name>_db.py` in `databases/` | `database.py`, `db.py` |
| Router files | `<resource>_router.py` | `<resource>.py`, `<resource>_routes.py` |
| Helper files | `<resource>_helpers.py` | `<resource>_service.py`, `<resource>_utils.py` |
| SQLAlchemy models | `User` (no suffix) | `UserModel`, `UserTable` |
| Pydantic schemas | `UserBase/Create/Patch/Response` | `UserSchema`, `UserIn`, `UserOut`, `UserUpdate` |
| URL paths | kebab-case, plural, no verbs | `/getUsers`, `/user/`, `/user/create` |

## Models — common violations

| Element | Required | Common violation |
|---|---|---|
| Model class name | `User` (no suffix) | `UserModel`, `UserTable` |
| `__tablename__` | plural snake_case: `"users"` | `"user"`, `"User"` |
| Base import | `from app.databases.backend_db import Base` | `Base = declarative_base()` in model file |
| Relationships | `back_populates=` on both sides | `backref=` |
| Timestamps | `created_at`/`updated_at` columns added manually | `TimestampMixin` |
| `models/__init__.py` | imports all models (required for Alembic autogenerate) | missing or empty |
| `__table_args__` | `{"schema": settings.DB_SCHEMA}` on every model | hard-coded schema string, `os.getenv()` directly |

## Routers — common violations

| Element | Required | Common violation |
|---|---|---|
| Path parameters | `Annotated[int, Path(description=..., examples=[...], ge=1)]` | bare `int`, or `id: int = Path(...)` without `Annotated` |
| Query parameters | `Annotated[str \| None, Query(default=None, description=..., examples=[...])]` | bare `str \| None = None`, or `= Query(...)` without `Annotated` |
| Constraints on params | `ge=1` on IDs, `min_length`/`max_length` on strings, `pattern` where format is known | unconstrained params |
| HTTP errors in helpers | never — helpers return `None` for not-found | `raise HTTPException` inside a helper |
| Commit location | `db.commit()` in router only | `db.commit()` inside a helper |
| IntegrityError handling | caught around `db.commit()` in POST/PATCH, `db.rollback()` then 409 | unhandled, or caught in helper |

## Folder structure — violations to flag

- `core/` folder present
- Service layer or repository layer present
- Feature-based layout instead of layer-based
- `migrations/` folder absent when Alembic is in use

## Settings — items not in SKILL.md

- [ ] Flat `Settings` class — no nested config classes
- [ ] `model_config = SettingsConfigDict(env_file=".env")` — not `class Config`
- [ ] `SERVICE_NAME` and `OPENAPI_VERSION` present — `OPENAPI_VERSION` defaults to `"3.1.0"`
- [ ] `DB_SCHEMA` present — never hard-coded in model files
- [ ] Six pool variables: `DB_POOL_SIZE`, `DB_MAX_OVERFLOW`, `DB_POOL_TIMEOUT`, `DB_POOL_RECYCLE`, `DB_POOL_PRE_PING`, `DB_ECHO`
- [ ] `settings` singleton imported — `Settings()` not re-instantiated elsewhere (except `conftest.py`)
- [ ] Every field uses `Annotated[type, Field(...)]` with `description`, `examples`, and constraints — never bare typed fields like `SERVICE_NAME: str`
