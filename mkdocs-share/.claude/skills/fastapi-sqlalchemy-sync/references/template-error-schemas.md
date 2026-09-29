# Template: schemas/output_schemas/error_schemas.py

One file per service — never duplicate error schemas per resource.
Every error schema uses `Annotated[type, Field(...)]` — never `constr`, `conlist`, or `confloat`.

```python
from typing import Annotated, Literal

from pydantic import BaseModel, Field


class BadRequestResponse(BaseModel):
    status_code: Literal[400]
    message: Annotated[str, Field(
        min_length=1,
        max_length=100,
        pattern=r'^[a-zA-Z-_.\s]*$',
        description="HTTP status message",
        examples=["Bad Request"],
    )]
    error: Annotated[str, Field(
        min_length=1,
        max_length=10000,
        pattern=r'^[\s\S]*$',
        description="Detailed error information",
        examples=["Cannot delete a project with active resources"],
    )]


class UnauthorizedResponse(BaseModel):
    status_code: Literal[401]
    message: Annotated[str, Field(
        default="Not authenticated",
        min_length=1,
        max_length=100,
        pattern=r'^[a-zA-Z\s]*$',
        description="HTTP status message",
        examples=["Not authenticated"],
    )]


class ForbiddenResponse(BaseModel):
    status_code: Literal[403]
    message: Annotated[str, Field(
        default="Forbidden",
        min_length=1,
        max_length=100,
        pattern=r'^[a-zA-Z\s]*$',
        description="HTTP status message",
        examples=["Forbidden"],
    )]
    error: Annotated[str, Field(
        min_length=1,
        max_length=10000,
        pattern=r'^[\s\S]*$',
        description="Reason access was denied",
        examples=["You do not have permission to perform this action"],
    )]


class NotFoundResponse(BaseModel):
    status_code: Literal[404]
    message: Annotated[str, Field(
        default="Data not found",
        min_length=1,
        max_length=100,
        pattern=r'^[a-zA-Z-_.\s]*$',
        description="HTTP status message",
        examples=["Data not found"],
    )]


class ConflictResponse(BaseModel):
    status_code: Literal[409]
    message: Annotated[str, Field(
        default="Already exists",
        min_length=1,
        max_length=100,
        pattern=r'^[a-zA-Z-_.\s]*$',
        description="HTTP status message",
        examples=["Already exists"],
    )]


class MethodNotAllowedResponse(BaseModel):
    status_code: Literal[405]
    message: Annotated[str, Field(
        default="Method Not Allowed",
        min_length=1,
        max_length=100,
        pattern=r'^[a-zA-Z\s]*$',
        description="HTTP status message",
        examples=["Method Not Allowed"],
    )]
    error: Annotated[str, Field(
        default="This HTTP method is not allowed for the requested endpoint.",
        min_length=1,
        max_length=100,
        pattern=r'^[a-zA-Z\s]*$',
        description="Error details",
        examples=["This HTTP method is not allowed for the requested endpoint."],
    )]


class ValidationErrorDetail(BaseModel):
    loc: Annotated[list[str | int], Field(
        description="Location of the validation error — path to the field",
        examples=[["body", "name"]],
    )]
    msg: Annotated[str, Field(
        min_length=1,
        max_length=10000,
        pattern=r'^[\s\S]*$',
        description="Validation error message",
        examples=["field required"],
    )]
    type: Annotated[str, Field(
        min_length=1,
        max_length=10000,
        pattern=r'^[\s\S]*$',
        description="Pydantic error type identifier",
        examples=["missing"],
    )]


class HTTPValidationError(BaseModel):
    status_code: Literal[422]
    detail: Annotated[list[ValidationErrorDetail], Field(
        max_length=50,
        description="List of field-level validation errors",
        examples=[[{"loc": ["body", "name"], "msg": "field required", "type": "missing"}]],
    )]


class InternalServerErrorResponse(BaseModel):
    status_code: Literal[500]
    message: Annotated[str, Field(
        default="Internal Server Error",
        min_length=1,
        max_length=100,
        pattern=r'^[a-zA-Z\s]*$',
        description="HTTP status message",
        examples=["Internal Server Error"],
    )]
    error: Annotated[str, Field(
        min_length=1,
        max_length=10000,
        pattern=r'^[\s\S]*$',
        description="Detailed error information",
        examples=["An unexpected error occurred"],
    )]
```

## Usage in routers

```python
from app.schemas.output_schemas.error_schemas import (
    BadRequestResponse,
    NotFoundResponse,
    ConflictResponse,
    MethodNotAllowedResponse,
    HTTPValidationError,
    InternalServerErrorResponse,
)

@router.post(
    "",
    response_model=ResourceResponse,
    status_code=201,
    summary="Create a resource",
    description="Creates a new resource. Name must be unique.",
    responses={
        201: {"description": "Resource created successfully"},
        400: {"model": BadRequestResponse, "description": "Bad request — business logic rejection"},
        404: {"model": NotFoundResponse, "description": "Resource not found"},
        405: {"model": MethodNotAllowedResponse, "description": "HTTP method not allowed"},
        409: {"model": ConflictResponse, "description": "A resource with this name already exists"},
        422: {"model": HTTPValidationError, "description": "Validation error in request body"},
        500: {"model": InternalServerErrorResponse, "description": "Unexpected server error"},
    },
)
```

Only include status codes that are actually reachable from the endpoint — don't add all codes to every endpoint.

## Status code reference

| Code | Schema | When to use | Who raises it |
|---|---|---|---|
| 200 | — | Successful GET, PUT, DELETE with body | Router returns response |
| 201 | — | Successful POST — resource created | `status_code=201` on endpoint |
| 400 | `BadRequestResponse` | Business logic rejection | `HTTPException(status_code=400)` in router |
| 401 | `UnauthorizedResponse` | Missing or invalid auth credentials | Auth middleware / dependency |
| 403 | `ForbiddenResponse` | Authenticated but not authorized | Auth dependency |
| 404 | `NotFoundResponse` | Resource not found | `HTTPException(status_code=404)` in router |
| 405 | `MethodNotAllowedResponse` | HTTP method not allowed | FastAPI auto-handles — never raise manually |
| 409 | `ConflictResponse` | Duplicate or unique constraint violation | `HTTPException(status_code=409)` in router |
| 422 | `HTTPValidationError` | Pydantic validation error | FastAPI/Pydantic auto-handles — never raise manually |
| 500 | `InternalServerErrorResponse` | Unexpected server error | Never raise manually |

## Rules

- Never use `constr`, `conlist`, or `confloat` — always use `Annotated[type, Field(...)]`
- Always include `pattern` on every `str` field — use `r'^[\s\S]*$'` for free-text fields with no character restriction
- Never raise `HTTPException(status_code=422)` manually — let Pydantic handle it
- Never raise `HTTPException(status_code=405)` manually — FastAPI handles it
- Never raise `HTTPException(status_code=500)` — fix the underlying bug instead
- **400 vs 422**: 422 is a Pydantic field-level error; 400 is a business logic rejection
- **401 vs 403**: 401 = "who are you?"; 403 = "I know who you are, but no"
- Only include reachable status codes in `responses=` — don't add all codes to every endpoint
