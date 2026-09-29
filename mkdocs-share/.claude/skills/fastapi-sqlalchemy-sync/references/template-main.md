# Template: app/main.py

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from sqlalchemy.exc import SQLAlchemyError

from app.routers.<resource>_router import router as <resource>_router
from app.schemas.output_schemas.error_schemas import InternalServerErrorResponse
from app.utils.logger import get_logger
# Import additional routers here

logger = get_logger()

openapi_tags = [
    {
        "name": "<resources>",
        "description": "Manage <resources> — create, list, retrieve, update, and delete.",
    },
    # Add one entry per router tag
]

app = FastAPI(
    title="<Service Name>",
    description="<One-line description of what this service does.>",
    version="1.0.0",
    openapi_version=settings.OPENAPI_VERSION,
    openapi_tags=openapi_tags,
    contact={
        "name": "<Team or owner name>",
        "email": "<contact email>",
    },
    docs_url="/docs",
    redoc_url="/redoc",
    openapi_url="/openapi.json",
    # servers=[
    #     {"url": "http://localhost:8000", "description": "Local"},
    #     {"url": "https://staging.example.com", "description": "Staging"},
    #     {"url": "https://api.example.com", "description": "Production"},
    # ],
    # Security schemes are configured via FastAPI dependencies (HTTPBearer, OAuth2, APIKey).
    # Uncomment and wire the appropriate scheme when authentication is required:
    # from fastapi.security import HTTPBearer
    # security = HTTPBearer()
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
    allow_credentials=False,
)


@app.exception_handler(SQLAlchemyError)
async def sqlalchemy_exception_handler(request: Request, exc: SQLAlchemyError):
    logger.error(f"Database error on {request.method} {request.url}: {exc}")
    return JSONResponse(
        status_code=500,
        content=InternalServerErrorResponse(
            status_code=500,
            message="Internal Server Error",
            error="A database error occurred. Please try again later.",
        ).model_dump(exclude={"status_code"}),
    )


@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unexpected error on {request.method} {request.url}: {exc}")
    return JSONResponse(
        status_code=500,
        content=InternalServerErrorResponse(
            status_code=500,
            message="Internal Server Error",
            error="An unexpected error occurred. Please try again later.",
        ).model_dump(exclude={"status_code"}),
    )


app.include_router(<resource>_router)
# Register additional routers here
```

## Rules

- `allow_origins=["*"]` is a development default — remind the user to restrict to specific origins in production
- `allow_credentials=False` when using `allow_origins=["*"]` — browsers block credentialed requests to wildcard origins
- Register every router with `app.include_router()` — no prefix here, prefix belongs on the `APIRouter()` in the router file
- Add one `openapi_tags` entry per router — `name` must exactly match the `tags=` value on the `APIRouter()`
- Always set `title`, `description`, and `version` on the `FastAPI()` constructor for enterprise-grade OpenAPI docs
- Never put `str(exc)` in the response body — log it, return a sanitized message to the client
- `SQLAlchemyError` handler catches DB and connection failures; `Exception` handler catches everything else
