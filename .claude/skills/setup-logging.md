---
name: setup-logging
description: Create structured logging with Rich console handler and uvicorn integration. Use when: (1) setting up logging for a new project, (2) user asks for logging configuration, (3) adding Rich console output, (4) configuring uvicorn logging.
---

# Setup Logging

Create logging configuration using Python's `dictConfig` with Rich console output.

## Gather Requirements

Use AskUserQuestion tool:

1. **Default Log Level**: INFO (Recommended), DEBUG, WARNING, ERROR
2. **Uvicorn Integration**: Yes (Recommended), No - configures uvicorn loggers to use Rich handler

## Directory Structure

```
telemetry/
├── __init__.py
└── logger.py
```

## Core Files

### telemetry/logger.py

```python
from logging.config import dictConfig

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
        "loggers": {
            "uvicorn": {"handlers": ["default"], "level": "INFO", "propagate": False},
            "uvicorn.error": {"handlers": ["default"], "level": "INFO", "propagate": False},
            "uvicorn.access": {"handlers": ["default"], "level": "INFO", "propagate": False},
        },
    })
```

**Adjustments:**
- Change `"level": "INFO"` to selected level (DEBUG/WARNING/ERROR)
- Remove `"loggers"` section if uvicorn integration not selected

### telemetry/__init__.py

```python
from .logger import setup_logging

__all__ = ["setup_logging"]
```

## Dependencies

```bash
pip install rich
```

## Usage

```python
from telemetry import setup_logging
import logging

setup_logging()
logger = logging.getLogger(__name__)

logger.info("Application started")
logger.error("Error occurred", exc_info=True)
```

**With uvicorn:**

```python
import uvicorn
from telemetry import setup_logging

if __name__ == "__main__":
    setup_logging()
    uvicorn.run("main:app", host="0.0.0.0", port=8000, log_config=None)
```

## Advanced Options (if requested)

**File logging:**

```python
"handlers": {
    "default": {"class": "rich.logging.RichHandler", "level": "INFO", "formatter": "default"},
    "file": {"class": "logging.FileHandler", "filename": "app.log", "level": "DEBUG", "formatter": "default"},
},
"root": {"handlers": ["default", "file"], "level": "DEBUG"},
```

**JSON logging (production):**

```python
"formatters": {
    "default": {"format": "[%(asctime)s] %(levelname)s - %(message)s", "datefmt": "%Y-%m-%d %H:%M:%S"},
    "json": {"class": "pythonjsonlogger.jsonlogger.JsonFormatter", "format": "%(asctime)s %(levelname)s %(name)s %(message)s"}
},
```

## Critical Notes

- Call `setup_logging()` **once** at startup before any logging calls
- `disable_existing_loggers: False` preserves third-party library loggers
- `propagate: False` on uvicorn loggers prevents duplicate messages
