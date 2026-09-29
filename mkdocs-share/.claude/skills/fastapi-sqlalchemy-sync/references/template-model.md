# Template: models/<resource>.py

One file per SQLAlchemy model. Name the file after the resource in singular snake_case (e.g., `project.py`).
Import `Base` from `app.databases.backend_db` — never define a second `Base`.

```python
from sqlalchemy import Column, Integer, String, DateTime, func

from app.databases.backend_db import Base
from app.config.settings import settings


class Project(Base):
    __tablename__ = "projects"
    __table_args__ = {"schema": settings.DB_SCHEMA}

    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False, unique=True)
    description = Column(String, nullable=True)
    created_at = Column(DateTime, server_default=func.now(), nullable=False)
    updated_at = Column(DateTime, server_default=func.now(), onupdate=func.now(), nullable=False)
```

## Rules

- Class name is singular PascalCase — no `Model` or `Table` suffix (e.g., `Project`, not `ProjectModel`)
- `__tablename__` is plural snake_case (e.g., `"projects"`, not `"project"`)
- Always import `Base` from `app.databases.backend_db` — never call `declarative_base()` again
- Always set `__table_args__ = {"schema": settings.DB_SCHEMA}` — never hard-code the schema name or use `os.getenv()` directly
- `nullable=False` on all required columns — never rely on DB defaults for required fields
- `server_default=func.now()` — the database sets `created_at` at insert time
- `onupdate=func.now()` — the database sets `updated_at` at every update
- Add `created_at` / `updated_at` manually per model when needed — no `TimestampMixin`
- Use `back_populates` on all relationships — never `backref`

## Common violations

| Violation | Correct |
|---|---|
| `class UserModel(Base)` | `class User(Base)` |
| `__tablename__ = "user"` | `__tablename__ = "users"` |
| `Base = declarative_base()` in model file | import `Base` from `app.databases.backend_db` |
| `backref="projects"` on relationship | `back_populates="projects"` on both sides |
| Timestamp mixin | add `created_at` / `updated_at` columns manually |
