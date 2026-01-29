---
name: setup-logging
description: Create structured logging with Rich console handler and uvicorn integration
---

# Setup Logging Skill

You are setting up structured logging for the user's project. This skill creates a logging configuration using Python's `dictConfig` with Rich console output for enhanced readability.

## Step 1: Gather Requirements

Ask the user these questions using the AskUserQuestion tool:

1. **Default Log Level** - "What should be the default log level?"
   - Options: INFO (Recommended), DEBUG, WARNING, ERROR
   - Description: INFO is suitable for production, DEBUG for development

2. **Uvicorn Integration** - "Include uvicorn logger configuration?"
   - Options: Yes (Recommended), No
   - Description: Configures uvicorn, uvicorn.error, and uvicorn.access loggers to use Rich handler

## Step 2: Create Directory Structure

Create the following directory structure:
```
telemetry/
├── __init__.py
└── logger.py           # Logging setup with Rich handler
```

## Step 3: Create Files

### telemetry/logger.py

**Base version (INFO level, with uvicorn):**
```python
from logging.config import dictConfig


def setup_logging():
    """
    Configure application logging with Rich console handler.

    This function should be called once at application startup,
    before any logging calls are made.
    """
    dictConfig({
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": {
            "default": {
                "format": "[%(asctime)s] %(levelname)s - %(message)s",
                "datefmt": "%Y-%m-%d %H:%M:%S"
            }
        },
        "handlers": {
            "default": {
                "class": "rich.logging.RichHandler",
                "level": "INFO",
                "formatter": "default",
            },
        },
        "root": {
            "handlers": ["default"],
            "level": "INFO",
        },
        "loggers": {
            "uvicorn": {
                "handlers": ["default"],
                "level": "INFO",
                "propagate": False
            },
            "uvicorn.error": {
                "handlers": ["default"],
                "level": "INFO",
                "propagate": False
            },
            "uvicorn.access": {
                "handlers": ["default"],
                "level": "INFO",
                "propagate": False
            },
        },
    })
```

**Adjustments based on user selections:**

If user selects **DEBUG** level:
- Change all `"level": "INFO"` to `"level": "DEBUG"`

If user selects **WARNING** level:
- Change all `"level": "INFO"` to `"level": "WARNING"`

If user selects **ERROR** level:
- Change all `"level": "INFO"` to `"level": "ERROR"`

If user selects **No** for uvicorn integration:
- Remove the entire `"loggers"` section:
```python
def setup_logging():
    dictConfig({
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": {
            "default": {
                "format": "[%(asctime)s] %(levelname)s - %(message)s",
                "datefmt": "%Y-%m-%d %H:%M:%S"
            }
        },
        "handlers": {
            "default": {
                "class": "rich.logging.RichHandler",
                "level": "INFO",
                "formatter": "default",
            },
        },
        "root": {
            "handlers": ["default"],
            "level": "INFO",
        },
    })
```

### telemetry/__init__.py
```python
from .logger import setup_logging

__all__ = ["setup_logging"]
```

## Step 4: Report Required Dependencies

Tell the user which packages to install:

```bash
pip install rich
```

## Step 5: Show Usage Example

**Basic usage in application startup:**
```python
from telemetry import setup_logging
import logging

# Call once at application startup
setup_logging()

# Get a logger for your module
logger = logging.getLogger(__name__)

# Use the logger
logger.info("Application started")
logger.debug("Debug message")
logger.warning("Warning message")
logger.error("Error occurred", exc_info=True)
```

**With FastAPI:**
```python
from fastapi import FastAPI
from telemetry import setup_logging
import logging

# Setup logging before creating the app
setup_logging()

logger = logging.getLogger(__name__)
app = FastAPI()

@app.on_event("startup")
async def startup():
    logger.info("Application startup complete")

@app.get("/")
async def root():
    logger.debug("Root endpoint called")
    return {"status": "ok"}
```

**With uvicorn programmatic start:**
```python
import uvicorn
from telemetry import setup_logging

if __name__ == "__main__":
    setup_logging()
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        log_config=None  # Disable uvicorn's default logging config
    )
```

## Rich Handler Features

The Rich logging handler provides:

- **Colored output** - Different colors for each log level
- **Syntax highlighting** - Automatic highlighting of Python objects, paths, URLs
- **Tracebacks** - Beautiful, readable exception tracebacks
- **Timestamps** - Clean timestamp formatting
- **Log level badges** - Visible level indicators

Example output:
```
[2024-01-15 10:30:45] INFO     - Application started
[2024-01-15 10:30:45] DEBUG    - Processing request for user_id=123
[2024-01-15 10:30:46] WARNING  - Rate limit approaching for API key
[2024-01-15 10:30:47] ERROR    - Database connection failed
```

## Advanced Configuration Options

If the user needs more customization, you can offer these additional options:

**Add file logging:**
```python
"handlers": {
    "default": {
        "class": "rich.logging.RichHandler",
        "level": "INFO",
        "formatter": "default",
    },
    "file": {
        "class": "logging.FileHandler",
        "filename": "app.log",
        "level": "DEBUG",
        "formatter": "default",
    },
},
"root": {
    "handlers": ["default", "file"],
    "level": "DEBUG",
},
```

**Add JSON logging for production:**
```python
"formatters": {
    "default": {
        "format": "[%(asctime)s] %(levelname)s - %(message)s",
        "datefmt": "%Y-%m-%d %H:%M:%S"
    },
    "json": {
        "class": "pythonjsonlogger.jsonlogger.JsonFormatter",
        "format": "%(asctime)s %(levelname)s %(name)s %(message)s"
    }
},
```

## Important Notes

- Call `setup_logging()` **once** at application startup, before any logging calls
- The `disable_existing_loggers: False` setting preserves third-party library loggers
- `propagate: False` on uvicorn loggers prevents duplicate log messages
- Rich handler automatically detects terminal capabilities and degrades gracefully
- For production deployments, consider adding a file handler or JSON formatter
