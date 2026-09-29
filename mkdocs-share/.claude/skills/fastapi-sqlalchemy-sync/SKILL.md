---
name: fastapi-sqlalchemy-sync
description: Generate and review FastAPI applications using synchronous SQLAlchemy ORM and Pydantic V2. Use this skill when building or modifying a FastAPI service — especially when `fastapi` appears in `requirements.txt` or `pyproject.toml`, or when the user asks to create endpoints, models, schemas, routers, helpers, or database connections in a FastAPI project.
compatibility: Python>=3.11, fastapi>=0.100, sqlalchemy>=2.0, pydantic>=2.0, pydantic-settings>=2.0, alembic>=1.13
allowed-tools: Read Edit Write Glob Grep
metadata:
  author: DineshAvula
  version: "1.1"
---

# FastAPI SQLAlchemy Sync

## Folder structure

```
<service-root>/
  app/
    databases/        # one <database_name>_db.py per DB connection
    models/           # one file per SQLAlchemy model; __init__.py imports all models
    routers/          # one <resource>_router.py per resource
    schemas/
      input_schemas/
      output_schemas/
    helpers/          # one <resource>_helpers.py per resource
    utils/            # logger.py only
    config/           # settings.py only
    main.py
  migrations/         # Alembic migrations folder
  alembic.ini
  Dockerfile
  requirements.txt
```

## Naming conventions

| Element | Convention |
|---|---|
| DB connection files | `<database_name>_db.py` in `databases/` |
| Router files | `<resource>_router.py` in `routers/` |
| Helper files | `<resource>_helpers.py` in `helpers/` |
| SQLAlchemy models | `User` (no `Model` suffix) |
| DB table names | `users` (plural snake_case) |
| Pydantic schemas | `UserBase`, `UserCreate`, `UserPatch`, `UserResponse` |
| URL paths | plural nouns, `kebab-case`, no trailing slash, no verbs — use `""` (not `"/"`) for collection root in router |
| Config file | `settings.py` in `config/` |
| Logger file | `logger.py` in `utils/` |

## Key rules

### Endpoints
- `def` only — never `async def` (sync SQLAlchemy blocks the event loop inside async)
- Every endpoint must have `tags`, `summary`, `response_model`, and explicit `status_code`
- `HTTPException` raised directly in routers — never inside helpers
- Helpers return `None` when a resource is not found; routers check and raise `HTTPException(status_code=404)`
- Catch `IntegrityError` around `db.commit()` in POST and PATCH endpoints — `db.rollback()` then raise `HTTPException(status_code=409)`
- Use `PATCH` for partial updates — never `PUT`; `<Resource>Patch` schema has all fields optional; helper uses `model_dump(exclude_unset=True)`
- Every path parameter must use `Annotated[type, Path(...)]` and every query parameter must use `Annotated[type, Query(...)]` — with `description`, `examples`, and constraints (`ge`, `le`, `min_length`, `max_length`, `pattern`) wherever applicable; never bare type annotations or inline `= Path()` / `= Query()` without `Annotated`

### URL path rules
- Plural nouns, `kebab-case`, no verbs in paths
- Collection root: always path `""` (not `"/"`) — `"/"` produces a trailing slash which violates REST conventions
- Single resource: `/{resource_id}` — never `/get/{resource_id}` or similar verb patterns
- Nested sub-resources: `/{resource_id}/sub-resources` — never use query params to express ownership
- HTTP method defines the action: `GET` read, `POST` create, `PATCH` partial update, `DELETE` remove — never use `PUT`

### Database
- SQLAlchemy sync ORM only — never `AsyncSession`, never `async_sessionmaker`, never Core `text()` with string formatting
- `Base(DeclarativeBase)` lives in `databases/backend_db.py` — all models import from there, never call `declarative_base()` again
- Every model must set `__table_args__ = {"schema": settings.DB_SCHEMA}` — never hard-code schema names
- `back_populates` on all relationships — never `backref`
- No `TimestampMixin` — add `created_at`/`updated_at` columns manually per model when needed

### Schemas
- Pydantic V2 only — never `from_orm`, `class Config`, `@validator`, `@root_validator`
- Every field must use `Annotated[type, Field(...)]` with `description`, `examples`, and constraints where applicable — applies to Pydantic schemas **and** `BaseSettings` fields in `settings.py`
- `ConfigDict(from_attributes=True)` on all response schemas
- Use `model_validate(obj)` — never `from_orm(obj)`

### Architecture
- Fat-router-with-helpers: router functions call plain `def` helpers from `helpers/<resource>_helpers.py`
- No service layer, no repository layer, no `core/` folder

## OpenAPI requirements

**`APIRouter()`** must include:
- `tags=` — use the resource name (e.g., `tags=["projects"]`)

**Every endpoint** must include:
- `summary=` — short one-line label shown in Swagger UI
- `description=` — fuller explanation of what the endpoint does and any constraints
- `response_model=` — always a Pydantic response schema, never an ORM model
- `status_code=` — explicit on every endpoint (200, 201, 204, etc.)
- `responses=` — document all expected error codes (404, 409, 422, etc.)

**`main.py`** must include:
- `title=`, `description=`, `version=`, `openapi_version=settings.OPENAPI_VERSION` on the `FastAPI()` constructor
- `contact=` — team name and email
- `docs_url=`, `redoc_url=`, `openapi_url=` — explicit paths
- `openapi_tags=` — one entry per router; `name` must exactly match the router's `tags=` value

## Security (OWASP)

- **SQL injection**: always use ORM queries — never `text()` with f-strings or string concatenation
- **Authentication**: never store plain-text passwords — use `passlib` or equivalent
- **Sensitive data exposure**: never include password, secret, or token fields in response schemas
- **Security misconfiguration**: `allow_origins=["*"]` is for development only — remind the user to restrict origins in production; always set `DB_ECHO=False` in production
- **Input validation**: always use Pydantic constraints (`min_length`, `max_length`, `ge`, `le`, `pattern`) on input schemas — never trust raw user input

## Reference files

Load these on demand — do not load all at once:

| When | Load |
|---|---|
| Creating or editing a DB connection file | `references/template-database.md` |
| Creating or editing a SQLAlchemy model | `references/template-model.md` |
| Creating or editing `config/settings.py` | `references/template-config.md` |
| Creating or editing Pydantic schemas | `references/template-schemas.md` |
| Creating the shared error schema or using `responses=` | `references/template-error-schemas.md` |
| Creating or editing a router or helpers file | `references/template-router.md` |
| Creating or editing `utils/logger.py` | `references/template-logger.md` |
| Creating or editing `main.py` | `references/template-main.md` |
| Creating or editing tests | `references/template-tests.md` |
| Setting up Alembic migrations | `references/template-alembic.md` |
| Reviewing existing code for compliance | `references/STANDARDS.md` |
