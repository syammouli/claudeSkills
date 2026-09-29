# Template: schemas/

Place input schemas in `schemas/input_schemas/` and output schemas in `schemas/output_schemas/`.
Follow the `<Resource>Base`, `<Resource>Create`, `<Resource>Update`, `<Resource>Response` naming pattern.

## Pydantic V2 — what to use

### Every field must use `Annotated` + `Field`

```python
from typing import Annotated
from pydantic import BaseModel, Field, ConfigDict

class ExampleCreate(BaseModel):
    name: Annotated[str, Field(
        min_length=1,
        max_length=100,
        description="Full name of the resource",
        examples=["Example Name"],
    )]
    age: Annotated[int, Field(
        ge=0,
        le=150,
        description="Age in years",
        examples=[30],
    )]
```

- Always provide `description` and `examples` on every field
- Always apply constraints wherever meaningful: `min_length`, `max_length`, `ge`, `le`, `pattern`
- Use `Enum` types for fields with a fixed set of valid values

### Pattern constraints

Always pair `pattern` with `min_length`/`max_length`. Apply whenever the field has a known safe character set — names, slugs, codes, identifiers.

### Response schemas must use `ConfigDict(from_attributes=True)`

```python
class ExampleResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: Annotated[int, Field(description="Primary key", examples=[1])]
    name: Annotated[str, Field(description="Full name", examples=["Example Name"])]
```

### Optional fields

Use `str | None` with `default=None` — never `Optional[str]` (Pydantic V1 style):

```python
class ExampleCreate(BaseModel):
    description: Annotated[str | None, Field(
        default=None,
        max_length=1000,
        description="Optional description",
        examples=["A short description"],
    )]
```

### Validators

Use `@field_validator` for single-field validation. Use `mode="before"` to normalize raw input
before type coercion; use `mode="after"` (default) to validate the already-typed value.

**`mode="before"` — normalize/strip raw input:**

```python
from pydantic import field_validator

class ExampleCreate(BaseModel):
    name: Annotated[str, Field(min_length=1, max_length=255, description="Name", examples=["my-project"])]

    @field_validator("name", mode="before")
    @classmethod
    def strip_name(cls, v: str) -> str:
        return v.strip()
```

**`mode="after"` — reject invalid values with a meaningful error:**

```python
class ExampleCreate(BaseModel):
    code: Annotated[str, Field(min_length=3, max_length=10, description="Short code", examples=["ABC"])]

    @field_validator("code", mode="after")
    @classmethod
    def code_must_be_uppercase(cls, v: str) -> str:
        if not v.isupper():
            raise ValueError("code must be all uppercase letters")
        return v
```

`raise ValueError(...)` inside any validator → FastAPI automatically returns `422 Unprocessable Entity`
with the error message in the response body. Never raise `HTTPException` inside a validator.

Use `@model_validator` for cross-field validation:

```python
from pydantic import model_validator

class ExampleCreate(BaseModel):
    password: Annotated[str, Field(min_length=8, description="Password", examples=["secret123"])]
    confirm_password: Annotated[str, Field(description="Must match password", examples=["secret123"])]

    @model_validator(mode="after")
    def passwords_must_match(self) -> "ExampleCreate":
        if self.password != self.confirm_password:
            raise ValueError("passwords do not match")
        return self
```

## Anti-patterns — never use

| Anti-pattern | Replacement |
|---|---|
| `class Config` | `model_config = ConfigDict(...)` |
| `@validator` | `@field_validator` |
| `@root_validator` | `@model_validator` |
| `from_orm(obj)` | `model_validate(obj)` |
| `Field(...)` without `Annotated` | `Annotated[type, Field(...)]` |
| Fields without `description` | Always add `description` and `examples` |
| `Optional[str]` | `str \| None` — use union syntax; `Optional` is Pydantic V1 style |
| `raise HTTPException` inside a validator | `raise ValueError(...)` — Pydantic converts it to `422 Unprocessable Entity` automatically |
| String fields with no constraints | Always add `min_length`, `max_length`, and `pattern` where the format is known |

## Error schemas

Use typed error schemas from `references/template-error-schemas.md` in router `responses=`.
Never define a generic `ErrorResponse` — use the specific typed schema for each status code.

## Security

- Never include password, secret, token, or hash fields in response schemas
- Always validate and constrain string inputs — never accept unbounded `str` on create/update schemas
