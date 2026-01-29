---
name: setup-database
description: Scaffold async SQLAlchemy 2.0+ database layer with singleton pattern and multi-backend support
---

# Setup Database Skill

You are setting up an async SQLAlchemy database layer for the user's project. This skill creates a production-ready database infrastructure with singleton engine pattern, multi-backend support, and FastAPI integration.

## Step 1: Gather Requirements

Ask the user these questions using the AskUserQuestion tool:

1. **Database Backend** - "Which database backend should I configure?"
   - Options: PostgreSQL (Recommended), MySQL, SQLite
   - Description: PostgreSQL uses asyncpg, MySQL uses aiomysql, SQLite uses aiosqlite

2. **FastAPI Integration** - "Include FastAPI dependency injection?"
   - Options: Yes (Recommended), No
   - Description: Creates `get_db_session()` dependency for request-scoped sessions

3. **Unit of Work Pattern** - "Include Unit of Work pattern for atomic transactions?"
   - Options: Yes (Recommended), No
   - Description: Provides `UnitOfWork` context manager for transactional boundaries

4. **Pool Settings** - "Use custom connection pool settings or defaults?"
   - Options: Use defaults (Recommended), Custom settings
   - Description: Defaults: pool_size=10, max_overflow=20, timeout=30s, recycle=1800s

## Step 2: Create Directory Structure

Create the following directory structure:
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

## Step 3: Create Files

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
    """Supported database types."""

    POSTGRES = "postgres"
    MYSQL = "mysql"
    SQLITE = "sqlite"


class DatabaseSettings(BaseSettings):
    """Database settings derived from environment variables."""

    model_config = SettingsConfigDict(env_prefix="DB_", case_sensitive=False, frozen=True)

    type: DatabaseType = Field(default=DatabaseType.POSTGRES)
    host: str = Field(default="localhost")
    port: int = Field(default=5432)
    name: str = Field(default="app")
    user: str = Field(default="app")
    password: str = Field(default="")

    # Connection Pool Settings
    pool_size: int = Field(default=10)
    max_overflow: int = Field(default=20)
    pool_timeout_seconds: int = Field(default=30)
    pool_recycle_seconds: int = Field(default=1800)

    @classmethod
    def from_env(cls) -> "DatabaseSettings":
        """Build settings from the configured environment."""
        return cls()
```

**Note:** Adjust the default `type` and `port` based on user's backend choice:
- PostgreSQL: `DatabaseType.POSTGRES`, port `5432`
- MySQL: `DatabaseType.MYSQL`, port `3306`
- SQLite: `DatabaseType.SQLITE`, port not needed

### db/core/interface.py
```python
from abc import ABC, abstractmethod
from typing import Any

from sqlalchemy.engine import URL
from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine

from ..configuration.settings import DatabaseSettings


class DBConnector(ABC):
    """
    Abstract interface for database connectors.

    Any new database backend must implement this class to ensure
    compatibility with the core engine factory.
    """

    def __init__(self, settings: DatabaseSettings) -> None:
        self.settings = settings

    @abstractmethod
    def get_url(self) -> URL:
        """Construct the SQLAlchemy URL for this connector."""
        pass

    def get_engine_args(self) -> dict[str, Any]:
        """
        Return engine-specific arguments (pooling, timeouts, etc.).

        Subclasses can override this to provide driver-specific options.
        """
        return {
            "pool_size": self.settings.pool_size,
            "max_overflow": self.settings.max_overflow,
            "pool_timeout": self.settings.pool_timeout_seconds,
            "pool_recycle": self.settings.pool_recycle_seconds,
            "pool_pre_ping": True,
        }

    def create_engine(self) -> AsyncEngine:
        """Create and return a configured SQLAlchemy AsyncEngine."""
        return create_async_engine(
            self.get_url(),
            **self.get_engine_args(),
        )
```

### db/connectors/postgres.py (if PostgreSQL selected)
```python
from sqlalchemy.engine import URL

from ..core.interface import DBConnector


class PostgresConnector(DBConnector):
    """
    Postgres connector implementation using asyncpg.

    Compatible with:
    - PostgreSQL
    - Supabase (via PostgreSQL interface)
    - Any other PostgreSQL-wire compatible database
    """

    def get_url(self) -> URL:
        """Create a SQLAlchemy URL for asyncpg."""
        return URL.create(
            drivername="postgresql+asyncpg",
            username=self.settings.user,
            password=self.settings.password,
            host=self.settings.host,
            port=self.settings.port,
            database=self.settings.name,
        )
```

### db/connectors/mysql.py (if MySQL selected)
```python
from sqlalchemy.engine import URL

from ..core.interface import DBConnector


class MySQLConnector(DBConnector):
    """MySQL connector implementation using aiomysql."""

    def get_url(self) -> URL:
        """Create a SQLAlchemy URL for aiomysql."""
        return URL.create(
            drivername="mysql+aiomysql",
            username=self.settings.user,
            password=self.settings.password,
            host=self.settings.host,
            port=self.settings.port,
            database=self.settings.name,
        )
```

### db/connectors/sqlite.py (if SQLite selected)
```python
from typing import Any
from sqlalchemy.engine import URL
from ..core.interface import DBConnector


class SQLiteConnector(DBConnector):
    """
    SQLite connector implementation using aiosqlite.

    Useful for local development and testing.
    """

    def get_url(self) -> URL:
        """
        Create a SQLAlchemy URL for aiosqlite.
        Note: self.settings.name is used as the database file path.
        """
        return URL.create(
            drivername="sqlite+aiosqlite",
            database=self.settings.name,
        )

    def get_engine_args(self) -> dict[str, Any]:
        return {
            "pool_pre_ping": True,
        }
```

### db/core/factory.py
Create based on selected backend(s). Example for PostgreSQL only:
```python
from ..connectors.postgres import PostgresConnector
from .interface import DBConnector
from ..configuration.settings import DatabaseSettings, DatabaseType


def get_connector(settings: DatabaseSettings) -> DBConnector:
    """
    Factory function to return the appropriate DB connector.

    Args:
        settings: The database settings object containing the 'type' field.

    Returns:
        DBConnector: An instantiated connector for the requested database type.

    Raises:
        ValueError: If the database type is not supported.
    """
    match settings.type:
        case DatabaseType.POSTGRES:
            return PostgresConnector(settings)
        case _:
            raise ValueError(f"Unsupported database type: {settings.type}")
```

Add additional cases for each selected backend.

### db/core/engine.py
```python
from sqlalchemy.ext.asyncio import AsyncEngine, AsyncSession, async_sessionmaker

from .factory import get_connector
from ..configuration.settings import DatabaseSettings
from ..configuration.types import AsyncSessionMaker

_engine: AsyncEngine | None = None
_sessionmaker: AsyncSessionMaker | None = None


def get_engine() -> AsyncEngine:
    """
    Return a singleton async engine configured for production use.

    The engine is lazily initialized on the first call using settings
    from the environment and the appropriate connector factory.
    """
    global _engine
    if _engine is None:
        settings = DatabaseSettings.from_env()
        connector = get_connector(settings)
        _engine = connector.create_engine()
    return _engine


def get_sessionmaker() -> AsyncSessionMaker:
    """
    Return a singleton sessionmaker bound to the shared engine.

    The sessionmaker produces 'AsyncSession' instances for database interactions.
    """
    global _sessionmaker
    if _sessionmaker is None:
        _sessionmaker = async_sessionmaker(
            bind=get_engine(),
            expire_on_commit=False,
            class_=AsyncSession,
        )
    return _sessionmaker


async def dispose_engine() -> None:
    """
    Close the engine and reset the pool.

    This should be called during application shutdown to ensure all connections
    are closed gracefully.
    """
    global _engine, _sessionmaker
    if _engine is not None:
        await _engine.dispose()
    _engine = None
    _sessionmaker = None
```

### db/session/dependencies.py (if FastAPI integration selected)
```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession
from ..core.engine import get_sessionmaker


async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    """
    FastAPI dependency that yields a single async session per request.

    This dependency manages the session lifecycle:
    1. Creates a new AsyncSession from the pool.
    2. Yields it to the request handler.
    3. Automatically closes the session after the request is finished.
    """
    sessionmaker = get_sessionmaker()
    async with sessionmaker() as session:
        yield session
```

### db/session/unit_of_work.py (if Unit of Work selected)
```python
from typing import Self
from sqlalchemy.ext.asyncio import AsyncSession
from ..core.engine import get_sessionmaker
from ..configuration.types import AsyncSessionMaker


class UnitOfWork:
    """
    Unit of Work pattern for managing transactional boundaries.

    This context manager ensures that all database operations within its scope
    are atomic. It automatically commits the transaction on success and rolls
    back on exception.

    Usage:
        async with UnitOfWork() as session:
            repo = UserRepository(session)
            await repo.create(user)
    """

    def __init__(self, sessionmaker: AsyncSessionMaker | None = None) -> None:
        self._sessionmaker = sessionmaker or get_sessionmaker()
        self.session: AsyncSession | None = None

    async def __aenter__(self) -> AsyncSession:
        """Open a new session and begin the transaction."""
        self.session = self._sessionmaker()
        return self.session

    async def __aexit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        traceback: object | None,
    ) -> None:
        """
        Handle transaction completion.

        - If an exception occurs (exc_type is not None), rollback.
        - If success, commit.
        - Always close the session.
        """
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
        """
        Explicitly begin a unit of work outside a context manager.
        Useful for when context managers are not applicable.
        """
        await self.__aenter__()
        return self

    async def end(self, exc: BaseException | None = None) -> None:
        """End a unit of work created via begin/end."""
        await self.__aexit__(type(exc) if exc else None, exc, None)
```

## Step 4: Create __init__.py Files

### db/__init__.py
```python
from .core.engine import get_engine, get_sessionmaker, dispose_engine
from .configuration.settings import DatabaseSettings, DatabaseType

__all__ = [
    "get_engine",
    "get_sessionmaker",
    "dispose_engine",
    "DatabaseSettings",
    "DatabaseType",
]
```

Add additional exports based on selected options.

## Step 5: Report Required Dependencies

Tell the user which packages to install based on their selections:

**Base dependencies (always needed):**
```bash
pip install sqlalchemy pydantic-settings
```

**Backend-specific:**
- PostgreSQL: `pip install asyncpg`
- MySQL: `pip install aiomysql`
- SQLite: `pip install aiosqlite`

## Step 6: Provide Environment Variable Template

```env
# Database Configuration
DB_TYPE=postgres  # or mysql, sqlite
DB_HOST=localhost
DB_PORT=5432
DB_NAME=app
DB_USER=app
DB_PASSWORD=your_password

# Pool Settings (optional - defaults shown)
DB_POOL_SIZE=10
DB_MAX_OVERFLOW=20
DB_POOL_TIMEOUT_SECONDS=30
DB_POOL_RECYCLE_SECONDS=1800
```

## Step 7: Show Usage Example

```python
# FastAPI endpoint with dependency injection
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from db.session.dependencies import get_db_session

@app.get("/users")
async def get_users(session: AsyncSession = Depends(get_db_session)):
    result = await session.execute(select(User))
    return result.scalars().all()

# Using Unit of Work for transactions
from db.session.unit_of_work import UnitOfWork

async def create_user_with_profile(user_data, profile_data):
    async with UnitOfWork() as session:
        user = User(**user_data)
        session.add(user)
        await session.flush()  # Get user.id

        profile = Profile(user_id=user.id, **profile_data)
        session.add(profile)
        # Auto-commits on context exit, rolls back on exception
```

## Important Notes

- The engine and sessionmaker are **singletons** - never create additional instances
- Always use `await` with database operations - the layer is fully async
- Call `dispose_engine()` during application shutdown for graceful cleanup
- For testing, use `dispose_engine()` between tests to reset state
