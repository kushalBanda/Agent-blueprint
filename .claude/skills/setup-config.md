---
name: setup-config
description: Set up environment-based configuration with CORS middleware and exception handlers for FastAPI. Use when: (1) initializing project configuration, (2) user asks for env variable setup, (3) adding CORS to FastAPI app, (4) standardizing error responses.
---

# Setup Config

Create environment-based configuration layer using `os.getenv()` pattern with optional CORS middleware and exception handlers.

## Gather Requirements

Use AskUserQuestion tool:

1. **Environment Variable Categories** (multiSelect: true): General (app name, URLs), AI APIs (OpenAI, Anthropic), GitHub integration
2. **CORS Middleware**: Yes (Recommended), No - creates `setup_cors()` function
3. **Exception Handlers**: Yes (Recommended), No - standardized JSON error responses

## Directory Structure

```
config/
├── __init__.py
├── env.py              # Environment class with os.getenv()
├── cors.py             # CORS middleware setup (if selected)
└── exceptions.py       # Exception handlers (if selected)
```

## Core Files

### config/env.py

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Environment:
    # Add sections based on user selections
```

**General:**
```python
    ENV = os.getenv("ENV")
    PRODUCT_NAME = os.getenv("PRODUCT_NAME")
    APP_URL = os.getenv("APP_URL")
    LANDING_URL = os.getenv("LANDING_URL")
```

**AI APIs:**
```python
    OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
    OPENAI_MODEL = os.getenv("OPENAI_MODEL")
    ANTHROPIC_API_KEY = os.getenv("ANTHROPIC_API_KEY")
    ANTHROPIC_MODEL = os.getenv("ANTHROPIC_MODEL")
```

**GitHub:**
```python
    GITHUB_TOKEN = os.getenv("GITHUB_TOKEN")
    GITHUB_BASE_URL = os.getenv("GITHUB_BASE_URL")
```

### config/cors.py (if selected)
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from config.env import Environment

def setup_cors(app: FastAPI):
    origins = list(filter(None, [
        Environment.APP_URL,
        Environment.LANDING_URL,
        "http://localhost:3000",
        "http://127.0.0.1:3000",
        "http://127.0.0.1:10000",
    ]))
    app.add_middleware(
        CORSMiddleware,
        allow_origins=origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
```

### config/exceptions.py (if selected)
```python
from fastapi import HTTPException, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import ValidationError
from pydantic_core import ValidationError as PydanticCoreValidationError
import logging

logger = logging.getLogger(__name__)

async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(status_code=exc.status_code, content={"message": exc.detail, "data": None})

async def validation_exception_handler(request: Request, exc: RequestValidationError):
    error_details = [{"field": " -> ".join(str(loc) for loc in e["loc"]), "message": e["msg"], "type": e["type"]} for e in exc.errors()]
    return JSONResponse(status_code=422, content={"message": "Validation error", "data": error_details, "error_count": len(error_details)})

async def pydantic_validation_exception_handler(request: Request, exc: ValidationError):
    logger.warning(f"Pydantic validation error on {request.url}: {exc}")
    error_details = [{"field": " -> ".join(str(loc) for loc in e["loc"]), "message": e["msg"], "type": e["type"], "input": e.get("input")} for e in exc.errors()]
    return JSONResponse(status_code=422, content={"message": "Data validation error", "data": error_details, "error_count": len(error_details)})

async def pydantic_core_validation_exception_handler(request: Request, exc: PydanticCoreValidationError):
    logger.warning(f"Pydantic core validation error on {request.url}: {exc}")
    error_details = [{"field": " -> ".join(str(loc) for loc in e["loc"]), "message": e["msg"], "type": e["type"], "input": e.get("input")} for e in exc.errors()]
    return JSONResponse(status_code=422, content={"message": "Data validation error", "data": error_details, "error_count": len(error_details)})

async def internal_server_error_handler(request: Request, exc: Exception):
    logger.error(f"Internal server error on {request.url}: {exc}", exc_info=True)
    return JSONResponse(status_code=500, content={"message": "Internal server error", "data": None, "error_type": type(exc).__name__})

exception_handlers = [
    (HTTPException, http_exception_handler),
    (RequestValidationError, validation_exception_handler),
    (ValidationError, pydantic_validation_exception_handler),
    (PydanticCoreValidationError, pydantic_core_validation_exception_handler),
    (Exception, internal_server_error_handler)
]
```

### config/__init__.py
```python
from .env import Environment

__all__ = ["Environment"]

# Add if CORS selected: from .cors import setup_cors
# Add if exceptions selected: from .exceptions import exception_handlers
```

## Dependencies

**Base:** `pip install python-dotenv`

**If CORS/exceptions:** `pip install fastapi pydantic`

## Environment Variables (.env.example)

```env
# General
ENV=development
PRODUCT_NAME=MyApp
APP_URL=http://localhost:3000
LANDING_URL=http://localhost:3000

# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4

# Anthropic
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-3-opus-20240229

# GitHub
GITHUB_TOKEN=ghp_...
GITHUB_BASE_URL=https://api.github.com
```

## Usage

```python
from fastapi import FastAPI
from config import Environment, setup_cors, exception_handlers

app = FastAPI(title=Environment.PRODUCT_NAME)
setup_cors(app)

for exc_class, handler in exception_handlers:
    app.add_exception_handler(exc_class, handler)

@app.get("/")
async def root():
    return {"app": Environment.PRODUCT_NAME, "environment": Environment.ENV}
```

## Critical Notes

- `Environment` class attributes are loaded at import time (read once)
- Mock `os.environ` before importing for testing
- CORS origins filter out `None` for unset environment variables
