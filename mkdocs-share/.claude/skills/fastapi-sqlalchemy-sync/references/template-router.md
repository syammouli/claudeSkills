# Template: routers/<resource>_router.py

One router file per resource. Router functions are thin — they call helpers, handle HTTP errors,
commit transactions, and return response schemas. Business logic lives in `helpers/<resource>_helpers.py`.

```python
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, Path, Query
from sqlalchemy.exc import IntegrityError
from sqlalchemy.orm import Session

from app.databases.backend_db import get_backend_db
from app.schemas.input_schemas.<resource>_schemas import <Resource>Create, <Resource>Patch
from app.schemas.output_schemas.<resource>_schemas import <Resource>Response
from app.schemas.output_schemas.error_schemas import BadRequestResponse, ConflictResponse, NotFoundResponse
from app.helpers.<resource>_helpers import (
    get_<resource>_by_id,
    get_all_<resources>,
    create_<resource>,
    patch_<resource>,
    delete_<resource>,
)

router = APIRouter(
    prefix="/<resources>",
    tags=["<resources>"],
)


@router.get(
    "",
    response_model=list[<Resource>Response],
    status_code=200,
    summary="List all <resources>",
    description="Returns all <resources>. Supports optional filtering via query parameters.",
    responses={
        200: {"description": "List of <resources> returned successfully"},
        400: {"model": BadRequestResponse, "description": "Invalid state or data integrity issue"},
        422: {"description": "Invalid query parameter value — FastAPI auto-generates this schema"},
    },
)
def list_<resources>(db: Session = Depends(get_backend_db)):
    return get_all_<resources>(db)


@router.get(
    "/{<resource>_id}",
    response_model=<Resource>Response,
    status_code=200,
    summary="Get a <resource> by ID",
    description="Fetches a single <resource> by its unique identifier.",
    responses={
        200: {"description": "<Resource> returned successfully"},
        400: {"model": BadRequestResponse, "description": "Invalid state or data integrity issue"},
        404: {"model": NotFoundResponse, "description": "<Resource> not found"},
    },
)
def get_<resource>(
    <resource>_id: Annotated[int, Path(description="<Resource> primary key", examples=[1], ge=1)],
    db: Session = Depends(get_backend_db),
):
    item = get_<resource>_by_id(db, <resource>_id)
    if item is None:
        raise HTTPException(status_code=404, detail=NotFoundResponse(status_code=404).model_dump(exclude={"status_code"}))
    return item


@router.post(
    "",
    response_model=<Resource>Response,
    status_code=201,
    summary="Create a new <resource>",
    description="Creates a new <resource>. The name must be unique.",
    responses={
        201: {"description": "<Resource> created successfully"},
        400: {"model": BadRequestResponse, "description": "Business rule prevents creation"},
        409: {"model": ConflictResponse, "description": "A <resource> with this name already exists"},
        422: {"description": "Validation error in request body — FastAPI auto-generates this schema"},
    },
)
def create_<resource>_endpoint(
    payload: <Resource>Create,
    db: Session = Depends(get_backend_db),
):
    item = create_<resource>(db, payload)
    try:
        db.commit()
    except IntegrityError:
        db.rollback()
        raise HTTPException(status_code=409, detail=ConflictResponse(status_code=409).model_dump(exclude={"status_code"}))
    db.refresh(item)
    return item


@router.patch(
    "/{<resource>_id}",
    response_model=<Resource>Response,
    status_code=200,
    summary="Partially update a <resource>",
    description="Updates one or more fields on an existing <resource>. Only provided fields are changed.",
    responses={
        200: {"description": "<Resource> updated successfully"},
        400: {"model": BadRequestResponse, "description": "Business rule prevents update"},
        404: {"model": NotFoundResponse, "description": "<Resource> not found"},
        409: {"model": ConflictResponse, "description": "A <resource> with this name already exists"},
        422: {"description": "Validation error in request body — FastAPI auto-generates this schema"},
    },
)
def patch_<resource>_endpoint(
    <resource>_id: Annotated[int, Path(description="<Resource> primary key", examples=[1], ge=1)],
    payload: <Resource>Patch,
    db: Session = Depends(get_backend_db),
):
    item = get_<resource>_by_id(db, <resource>_id)
    if item is None:
        raise HTTPException(status_code=404, detail=NotFoundResponse(status_code=404).model_dump(exclude={"status_code"}))
    updated = patch_<resource>(db, item, payload)
    try:
        db.commit()
    except IntegrityError:
        db.rollback()
        raise HTTPException(status_code=409, detail=ConflictResponse(status_code=409).model_dump(exclude={"status_code"}))
    db.refresh(updated)
    return updated


@router.delete(
    "/{<resource>_id}",
    status_code=204,
    summary="Delete a <resource>",
    description="Permanently deletes a <resource> by its unique identifier.",
    responses={
        204: {"description": "<Resource> deleted successfully"},
        400: {"model": BadRequestResponse, "description": "Business rule prevents deletion"},
        404: {"model": NotFoundResponse, "description": "<Resource> not found"},
    },
)
def delete_<resource>_endpoint(
    <resource>_id: Annotated[int, Path(description="<Resource> primary key", examples=[1], ge=1)],
    db: Session = Depends(get_backend_db),
):
    item = get_<resource>_by_id(db, <resource>_id)
    if item is None:
        raise HTTPException(status_code=404, detail=NotFoundResponse(status_code=404).model_dump(exclude={"status_code"}))
    delete_<resource>(db, item)
    db.commit()
```

## URL path standards

| Rule | Correct | Wrong |
|---|---|---|
| Plural nouns | `/projects` | `/project` |
| kebab-case for multi-word resources | `/project-types` | `/projectTypes`, `/project_types` |
| No verbs in paths | `DELETE /projects/{id}` | `POST /projects/{id}/delete` |
| No trailing slash on collection root | path `""` (not `"/"`) on `APIRouter` | `"/"` |
| Single resource by ID | `/{resource_id}` | `/get/{resource_id}` |
| Nested sub-resources | `/projects/{project_id}/environments` | `/environments?projectId=x` |
| HTTP method defines the action | `GET` = read, `POST` = create, `PATCH` = partial update, `DELETE` = remove | Using `POST` for everything |

**Collection root path:** always use `""` (empty string), not `"/"`. With `prefix="/projects"`:
- `""` → `/projects` (correct — no trailing slash)
- `"/"` → `/projects/` (wrong — trailing slash)

## Rules

- All endpoints are `def` — never `async def`
- Every path parameter must use `Annotated[type, Path(...)]` and every query parameter must use `Annotated[type, Query(...)]` — with `description`, `examples`, and applicable constraints (`ge`, `le`, `min_length`, `max_length`, `pattern`); never bare type annotations or inline `= Path()` / `= Query()` without `Annotated`
- `db.commit()` and `db.refresh()` belong in the router, never in helpers
- `HTTPException` is raised in the router only — helpers return `None` for not-found, never raise HTTP errors
- Always pass `model_dump(exclude={"status_code"})` of the typed error schema as `detail` — never a bare string; the HTTP status code is already in the response header, omit it from the body
- Catch `IntegrityError` around `db.commit()` in POST and PATCH endpoints — call `db.rollback()` then raise `HTTPException(409)`; never rely on the global `SQLAlchemyError` handler for uniqueness violations
- Raise `400` for business logic rejections using `BadRequestResponse` — never for Pydantic validation errors (422) or uniqueness violations (409):
  ```python
  raise HTTPException(
      status_code=400,
      detail=BadRequestResponse(
          status_code=400,
          message="Bad Request",
          error="<Specific reason why the operation was rejected>",
      ).model_dump(exclude={"status_code"}),
  )
  ```
- `response_model` is always a Pydantic response schema — never an ORM model
- `DELETE` endpoints use `status_code=204` — no `response_model`, no return body

## Helper pattern

Helpers are plain `def` functions in `helpers/<resource>_helpers.py`:

```python
from sqlalchemy.orm import Session
from app.models.<resource> import <Resource>
from app.schemas.input_schemas.<resource>_schemas import <Resource>Create, <Resource>Patch


def get_<resource>_by_id(db: Session, <resource>_id: int) -> <Resource> | None:
    return db.query(<Resource>).filter(<Resource>.id == <resource>_id).first()


def get_all_<resources>(db: Session) -> list[<Resource>]:
    return db.query(<Resource>).all()


def create_<resource>(db: Session, payload: <Resource>Create) -> <Resource>:
    item = <Resource>(**payload.model_dump())
    db.add(item)
    db.flush()
    return item


def patch_<resource>(db: Session, item: <Resource>, payload: <Resource>Patch) -> <Resource>:
    for field, value in payload.model_dump(exclude_unset=True).items():
        setattr(item, field, value)
    db.flush()
    return item


def delete_<resource>(db: Session, item: <Resource>) -> None:
    db.delete(item)
    db.flush()
```

- `db.flush()` in helpers — stages changes without committing; the router commits
- Never call `db.commit()` inside a helper
- `exclude_unset=True` in `patch_<resource>` — only fields the client explicitly sent are applied
- Return the ORM object from helpers — the router converts to response schema via `response_model`

## Schema pattern for PATCH

`<Resource>Patch` must have all fields optional — client sends only the fields to change:

```python
class <Resource>Patch(BaseModel):
    name: Annotated[str | None, Field(
        default=None,
        min_length=1,
        max_length=100,
        description="Updated name",
        examples=["New Name"],
    )]
    description: Annotated[str | None, Field(
        default=None,
        max_length=1000,
        description="Updated description",
        examples=["A new description"],
    )]
```
