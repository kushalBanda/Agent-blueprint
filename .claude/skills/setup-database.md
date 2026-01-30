---
name: setup-database
description: Scaffold async SQLAlchemy 2.0+ database layer with singleton pattern and multi-backend support. Use when: (1) setting up a new database layer, (2) user asks for database configuration, (3) adding async DB support to a project, (4) configuring PostgreSQL/MySQL/SQLite with FastAPI.
---

# Setup Database

Scaffold production-ready async SQLAlchemy database infrastructure with singleton engine, multi-backend support, and FastAPI integration.

## Gather Requirements

Use AskUserQuestion tool:

1. **Database Backend**: PostgreSQL (Recommended), MySQL, SQLite
2. **FastAPI Integration**: Yes (Recommended), No - creates `get_db_session()` dependency
3. **Unit of Work Pattern**: Yes (Recommended), No - provides transactional `UnitOfWork` context manager
4. **Pool Settings**: Defaults (Recommended), Custom - defaults: pool_size=10, max_overflow=20

## Directory Structure

```
db/
├── __init__.py
├── configuration/
│   ├── __init__.py
│   ├── settings.py      # Pydantic Settings with DB_ prefix
│   └── types.py         # Type aliases
├── core/
│   ├── __init__.py
│   ├── engine.py        # Singleton engine and sessionmaker
│   ├── interface.py     # Abstract DBConnector base class
│   └── factory.py       # Connector factory function
├── connectors/
│   ├── __init__.py
│   └── [backend].py     # Selected backend connector(s)
└── session/
    ├── __init__.py
    ├── dependencies.py  # FastAPI dependency (if selected)
    └── unit_of_work.py  # UnitOfWork class (if selected)
```

## Core Files

### db/configuration/types.py
```python
from collections.abc import Callable
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

AsyncSessionMaker = async_sessionmaker[AsyncSession]
AsyncSessionFactory = Callable[[], AsyncSession]
```

### db/configuration/settings.py
```python
from enum import Enum
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict

class DatabaseType(str, Enum):
    POSTGRES = "postgres"
    MYSQL = "mysql"
    SQLITE = "sqlite"

class DatabaseSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="DB_", case_sensitive=False, frozen=True)

    type: DatabaseType = Field(default=DatabaseType.POSTGRES)
    host: str = Field(default="localhost")
    port: int = Field(default=5432)
    name: str = Field(default="app")
    user: str = Field(default="app")
    password: str = Field(default="")
    pool_size: int = Field(default=10)
    max_overflow: int = Field(default=20)
    pool_timeout_seconds: int = Field(default=30)
    pool_recycle_seconds: int = Field(default=1800)

    @classmethod
    def from_env(cls) -> "DatabaseSettings":
        return cls()
```

Adjust default `type` and `port` based on backend: PostgreSQL=5432, MySQL=3306, SQLite=N/A.

### db/core/interface.py
```python
from abc import ABC, abstractmethod
from typing import Any
from sqlalchemy.engine import URL
from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine
from ..configuration.settings import DatabaseSettings

class DBConnector(ABC):
    def __init__(self, settings: DatabaseSettings) -> None:
        self.settings = settings

    @abstractmethod
    def get_url(self) -> URL:
        pass

    def get_engine_args(self) -> dict[str, Any]:
        return {
            "pool_size": self.settings.pool_size,
            "max_overflow": self.settings.max_overflow,
            "pool_timeout": self.settings.pool_timeout_seconds,
            "pool_recycle": self.settings.pool_recycle_seconds,
            "pool_pre_ping": True,
        }

    def create_engine(self) -> AsyncEngine:
        return create_async_engine(self.get_url(), **self.get_engine_args())
```

### db/connectors/postgres.py
```python
from sqlalchemy.engine import URL
from ..core.interface import DBConnector

class PostgresConnector(DBConnector):
    def get_url(self) -> URL:
        return URL.create(
            drivername="postgresql+asyncpg",
            username=self.settings.user,
            password=self.settings.password,
            host=self.settings.host,
            port=self.settings.port,
            database=self.settings.name,
        )
```

### db/connectors/mysql.py
```python
from sqlalchemy.engine import URL
from ..core.interface import DBConnector

class MySQLConnector(DBConnector):
    def get_url(self) -> URL:
        return URL.create(
            drivername="mysql+aiomysql",
            username=self.settings.user,
            password=self.settings.password,
            host=self.settings.host,
            port=self.settings.port,
            database=self.settings.name,
        )
```

### db/connectors/sqlite.py
```python
from typing import Any
from sqlalchemy.engine import URL
from ..core.interface import DBConnector

class SQLiteConnector(DBConnector):
    def get_url(self) -> URL:
        return URL.create(drivername="sqlite+aiosqlite", database=self.settings.name)

    def get_engine_args(self) -> dict[str, Any]:
        return {"pool_pre_ping": True}
```

### db/core/factory.py
```python
from ..connectors.postgres import PostgresConnector
from .interface import DBConnector
from ..configuration.settings import DatabaseSettings, DatabaseType

def get_connector(settings: DatabaseSettings) -> DBConnector:
    match settings.type:
        case DatabaseType.POSTGRES:
            return PostgresConnector(settings)
        case _:
            raise ValueError(f"Unsupported database type: {settings.type}")
```

Add cases for each selected backend.

### db/core/engine.py
```python
from sqlalchemy.ext.asyncio import AsyncEngine, AsyncSession, async_sessionmaker
from .factory import get_connector
from ..configuration.settings import DatabaseSettings
from ..configuration.types import AsyncSessionMaker

_engine: AsyncEngine | None = None
_sessionmaker: AsyncSessionMaker | None = None

def get_engine() -> AsyncEngine:
    global _engine
    if _engine is None:
        settings = DatabaseSettings.from_env()
        connector = get_connector(settings)
        _engine = connector.create_engine()
    return _engine

def get_sessionmaker() -> AsyncSessionMaker:
    global _sessionmaker
    if _sessionmaker is None:
        _sessionmaker = async_sessionmaker(bind=get_engine(), expire_on_commit=False, class_=AsyncSession)
    return _sessionmaker

async def dispose_engine() -> None:
    global _engine, _sessionmaker
    if _engine is not None:
        await _engine.dispose()
    _engine = None
    _sessionmaker = None
```

### db/session/dependencies.py (if FastAPI selected)
```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession
from ..core.engine import get_sessionmaker

async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    sessionmaker = get_sessionmaker()
    async with sessionmaker() as session:
        yield session
```

### db/session/unit_of_work.py (if UoW selected)
```python
from typing import Self
from sqlalchemy.ext.asyncio import AsyncSession
from ..core.engine import get_sessionmaker
from ..configuration.types import AsyncSessionMaker

class UnitOfWork:
    def __init__(self, sessionmaker: AsyncSessionMaker | None = None) -> None:
        self._sessionmaker = sessionmaker or get_sessionmaker()
        self.session: AsyncSession | None = None

    async def __aenter__(self) -> AsyncSession:
        self.session = self._sessionmaker()
        return self.session

    async def __aexit__(self, exc_type, exc, traceback) -> None:
        if self.session is None:
            return
        try:
            if exc_type is None:
                await self.session.commit()
            else:
                await self.session.rollback()
        finally:
            await self.session.close()

    async def begin(self) -> Self:
        await self.__aenter__()
        return self

    async def end(self, exc: BaseException | None = None) -> None:
        await self.__aexit__(type(exc) if exc else None, exc, None)
```

### db/__init__.py
```python
from .core.engine import get_engine, get_sessionmaker, dispose_engine
from .configuration.settings import DatabaseSettings, DatabaseType

__all__ = ["get_engine", "get_sessionmaker", "dispose_engine", "DatabaseSettings", "DatabaseType"]
```

## Dependencies

**Base:** `pip install sqlalchemy pydantic-settings`

**Backend-specific:** PostgreSQL: `asyncpg` | MySQL: `aiomysql` | SQLite: `aiosqlite`

## Environment Variables

```env
DB_TYPE=postgres
DB_HOST=localhost
DB_PORT=5432
DB_NAME=app
DB_USER=app
DB_PASSWORD=your_password
DB_POOL_SIZE=10
DB_MAX_OVERFLOW=20
DB_POOL_TIMEOUT_SECONDS=30
DB_POOL_RECYCLE_SECONDS=1800
```

## Usage

```python
# FastAPI endpoint
@app.get("/users")
async def get_users(session: AsyncSession = Depends(get_db_session)):
    result = await session.execute(select(User))
    return result.scalars().all()

# Unit of Work
async def create_user_with_profile(user_data, profile_data):
    async with UnitOfWork() as session:
        user = User(**user_data)
        session.add(user)
        await session.flush()
        profile = Profile(user_id=user.id, **profile_data)
        session.add(profile)
```

## Critical Notes

- Engine and sessionmaker are **singletons** - do not create additional instances
- Call `dispose_engine()` during shutdown and between tests
