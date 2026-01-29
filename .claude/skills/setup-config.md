---
name: setup-config
description: Set up environment-based configuration with CORS middleware and exception handlers for FastAPI
---

# Setup Config Skill

You are setting up environment-based configuration for the user's project. This skill creates a clean configuration layer using `os.getenv()` pattern with optional CORS middleware and exception handlers for FastAPI.

## Step 1: Gather Requirements

Ask the user these questions using the AskUserQuestion tool:

1. **Environment Variable Categories** - "Which environment variable categories do you need?" (multiSelect: true)
   - Options: General (app name, URLs), AI APIs (OpenAI, Anthropic), GitHub integration
   - Description: Select all categories that apply to your project

2. **CORS Middleware** - "Include CORS middleware setup for FastAPI?"
   - Options: Yes (Recommended), No
   - Description: Creates `setup_cors()` function with common local development origins

3. **Exception Handlers** - "Include FastAPI exception handlers?"
   - Options: Yes (Recommended), No
   - Description: Creates standardized JSON error responses for HTTP, validation, and server errors

## Step 2: Create Directory Structure

Create the following directory structure:
```
config/
├── __init__.py
├── env.py              # Environment class with os.getenv()
├── cors.py             # CORS middleware setup (if selected)
└── exceptions.py       # Exception handlers (if selected)
```

## Step 3: Create Files

### config/env.py

Base template (always include):
```python
# config.py
import os
from dotenv import load_dotenv

# Load environment variables from .env
load_dotenv()


class Environment:
```

Add sections based on user's category selections:

**General (if selected):**
```python
    # General
    ENV = os.getenv("ENV")
    PRODUCT_NAME = os.getenv("PRODUCT_NAME")
    APP_URL = os.getenv("APP_URL")
    LANDING_URL = os.getenv("LANDING_URL")
```

**AI APIs (if selected):**
```python
    # OpenAI
    OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
    OPENAI_MODEL = os.getenv("OPENAI_MODEL")

    # Anthropic
    ANTHROPIC_API_KEY = os.getenv("ANTHROPIC_API_KEY")
    ANTHROPIC_MODEL = os.getenv("ANTHROPIC_MODEL")
```

**GitHub (if selected):**
```python
    # GitHub
    GITHUB_TOKEN = os.getenv("GITHUB_TOKEN")
    GITHUB_BASE_URL = os.getenv("GITHUB_BASE_URL")
```

### config/cors.py (if CORS middleware selected)
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from config.env import Environment


def setup_cors(app: FastAPI):
    """
    Configure CORS middleware for the FastAPI application.

    Allows requests from configured app URLs and common local development origins.
    """
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

### config/exceptions.py (if exception handlers selected)
```python
from fastapi import HTTPException, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import ValidationError
from pydantic_core import ValidationError as PydanticCoreValidationError
import logging

logger = logging.getLogger(__name__)


async def http_exception_handler(request: Request, exc: HTTPException):
    """Handle HTTP exceptions with standardized JSON response."""
    return JSONResponse(
        status_code=exc.status_code,
        content={"message": exc.detail, "data": None}
    )


async def validation_exception_handler(request: Request, exc: RequestValidationError):
    """Handle FastAPI request validation errors."""
    error_details = []
    for error in exc.errors():
        field_name = " -> ".join(str(loc) for loc in error["loc"])
        error_details.append({
            "field": field_name,
            "message": error["msg"],
            "type": error["type"]
        })

    return JSONResponse(
        status_code=422,
        content={
            "message": "Validation error",
            "data": error_details,
            "error_count": len(error_details)
        }
    )


async def pydantic_validation_exception_handler(request: Request, exc: ValidationError):
    """Handle Pydantic model validation errors."""
    logger.warning(f"Pydantic validation error on {request.url}: {exc}")

    error_details = []
    for error in exc.errors():
        field_name = " -> ".join(str(loc) for loc in error["loc"])
        error_details.append({
            "field": field_name,
            "message": error["msg"],
            "type": error["type"],
            "input": error.get("input")
        })

    return JSONResponse(
        status_code=422,
        content={
            "message": "Data validation error",
            "data": error_details,
            "error_count": len(error_details)
        }
    )


async def pydantic_core_validation_exception_handler(request: Request, exc: PydanticCoreValidationError):
    """Handle Pydantic core validation errors."""
    logger.warning(f"Pydantic core validation error on {request.url}: {exc}")

    error_details = []
    for error in exc.errors():
        field_name = " -> ".join(str(loc) for loc in error["loc"])
        error_details.append({
            "field": field_name,
            "message": error["msg"],
            "type": error["type"],
            "input": error.get("input")
        })

    return JSONResponse(
        status_code=422,
        content={
            "message": "Data validation error",
            "data": error_details,
            "error_count": len(error_details)
        }
    )


async def internal_server_error_handler(request: Request, exc: Exception):
    """Handle unexpected server errors."""
    logger.error(f"Internal server error on {request.url}: {exc}", exc_info=True)

    # Don't expose internal error details in production
    return JSONResponse(
        status_code=500,
        content={
            "message": "Internal server error",
            "data": None,
            "error_type": type(exc).__name__
        }
    )


# List of exception handlers to register with FastAPI
# Usage: for exc_class, handler in exception_handlers:
#            app.add_exception_handler(exc_class, handler)
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
```

Add additional exports based on selected options:
```python
# If CORS selected
from .cors import setup_cors

# If exceptions selected
from .exceptions import exception_handlers
```

## Step 4: Report Required Dependencies

Tell the user which packages to install:

**Base dependencies (always needed):**
```bash
pip install python-dotenv
```

**If CORS or exceptions selected:**
```bash
pip install fastapi pydantic
```

## Step 5: Generate .env.example Template

Create a `.env.example` file based on selected categories:

```env
# Environment Configuration
# Copy this file to .env and fill in your values
```

**General (if selected):**
```env
# General
ENV=development
PRODUCT_NAME=MyApp
APP_URL=http://localhost:3000
LANDING_URL=http://localhost:3000
```

**AI APIs (if selected):**
```env
# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4

# Anthropic
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-3-opus-20240229
```

**GitHub (if selected):**
```env
# GitHub
GITHUB_TOKEN=ghp_...
GITHUB_BASE_URL=https://api.github.com
```

## Step 6: Show Usage Example

```python
from fastapi import FastAPI
from config import Environment, setup_cors, exception_handlers

app = FastAPI(title=Environment.PRODUCT_NAME)

# Setup CORS middleware
setup_cors(app)

# Register exception handlers
for exc_class, handler in exception_handlers:
    app.add_exception_handler(exc_class, handler)

# Use environment variables
@app.get("/")
async def root():
    return {
        "app": Environment.PRODUCT_NAME,
        "environment": Environment.ENV
    }
```

## Important Notes

- The `Environment` class uses class-level attributes loaded at import time
- Values are read once when the module is first imported
- For testing, you can mock `os.environ` before importing the module
- The `exception_handlers` list pattern allows easy registration with any FastAPI app
- CORS origins include `None` filtering to handle unset environment variables
