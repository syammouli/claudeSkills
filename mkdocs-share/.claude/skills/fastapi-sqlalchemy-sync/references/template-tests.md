# Template: tests/

## conftest.py — PostgreSQL via .env.test

The test suite runs against a real PostgreSQL database. Load settings from `.env.test` by
instantiating `Settings` with a different env file. Drop and recreate all tables before each
test module to ensure a clean state.

### What conftest.py must do

1. Load `Settings(_env_file=".env.test")` to get the test database URL
2. Create a test engine from that URL (same pool settings as production)
3. Call `Base.metadata.drop_all(engine)` then `Base.metadata.create_all(engine)` at the start of each test (or test module, depending on desired isolation)
4. Provide a `db` fixture that yields a `Session` bound to the test engine
5. Provide a `client` fixture that creates a `TestClient` with `get_backend_db` overridden to use the test `db`

### Overriding the DB dependency

```python
from fastapi.testclient import TestClient
from app.main import app
from app.databases.backend_db import get_backend_db

def override_get_backend_db():
    # yield the test session here
    ...

app.dependency_overrides[get_backend_db] = override_get_backend_db
client = TestClient(app)
```

## Writing router tests

### Rules

- Always assert `response.status_code` before asserting the response body
- Test the happy path first, then 404 and validation errors (422)
- Use the `client` fixture for HTTP calls — never call router functions directly
- Use the `db` fixture to set up preconditions and verify side effects
- Never share state between tests — each test must be fully self-contained
- Name tests `test_<verb>_<resource>_<scenario>` (e.g., `test_create_user_success`, `test_get_user_not_found`)

### What to test per endpoint

| Endpoint | Required test cases |
|---|---|
| `GET /` | returns 200, returns list (empty and non-empty) |
| `GET /{id}` | returns 200 with correct data; returns 404 when not found |
| `POST /` | returns 201 with created resource; returns 422 on invalid input |
| `PUT /{id}` | returns 200 with updated data; returns 404 when not found |
| `DELETE /{id}` | returns 204; returns 404 when not found |

### Assertions

- Assert status code first: `assert response.status_code == 201`
- Then assert body fields: `assert response.json()["name"] == "expected"`
- For 204 responses: assert no body — `assert response.content == b""`
- For 422 responses: assert `response.json()["detail"]` is present

## Environment

- `.env.test` must define `DATABASE_URL` pointing to a dedicated test database
- Never run tests against the production or development database
- The test database name should clearly indicate it is for testing (e.g., `myapp_test`)
