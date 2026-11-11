# Async Python, Postgres, and SQLAlchemy

The combination of FastAPI, Async SQLAlchemy, and Alembic allows developers to build scalable, non-blocking Python backends with efficient database interactions and version-controlled schema management.

Here are examples illustrating the use of these tools in an asynchronous Python context:

## 1. Asynchronous SQLAlchemy Setup and Integration with FastAPI

To utilize asynchronous capabilities, SQLAlchemy requires an async engine and session, typically configured using a session manager pattern.

### Setting up the Asynchronous Engine and Session

The asynchronous engine is initialized using `create_async_engine`, and the database URL must specify an asynchronous driver, such as `asyncpg` for PostgreSQL (e.g., `postgresql+asyncpg://...`).

A **SessionManager** class can handle this initialization and resource management, using `AsyncEngine` and `async_sessionmaker`:

``` python
from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import declarative_base

# Initialize Engine (must use an async driver, e.g., postgresql+asyncpg)
# poolclass=AsyncAdaptedQueuePool supports asynchronous connection pooling
engine: AsyncEngine = create_async_engine(
    "postgresql+asyncpg://user:pass@host:port/db_name",
    echo=True, # Logs generated SQL queries
    # other parameters like pool_size, max_overflow
)

# Initialize Session Factory
async_session_maker: async_sessionmaker[AsyncSession] = async_sessionmaker(
    engine,
    expire_on_commit=False, # Keep ORM objects alive after commit
    autoflush=False,
    class_=AsyncSession,
)
```

### Using Dependency Injection in FastAPI

FastAPI leverages **dependency injection** to provide an `AsyncSession` instance to route functions. This pattern ensures that a new session is created for each request and properly yielded or closed upon completion.

An asynchronous generator function is used to manage the session lifecycle:

``` python
from typing import Annotated, AsyncGenerator
from fastapi import Depends
# Assuming 'sessionmanager' or 'async_session_maker' is globally accessible

async def get_async_db_session() -> AsyncGenerator[AsyncSession, None]:
    """Yields a database session."""
    # This structure ensures commit/rollback/closing logic is handled
    async with async_session_maker() as session:
        # Optional: Set schema path if using multi-tenancy
        # await session.execute(text("SET search_path TO {schema}"))
        try:
            yield session
        except Exception as e:
            await session.rollback()
            raise RuntimeError(f"Database session error: {e!r}") from e
```

In a FastAPI route, this session is injected using `Annotated` and `Depends`:

``` python
from fastapi import APIRouter
# Using Annotated for dependency injection
DB_Session = Annotated[AsyncSession, Depends(get_async_db_session)]
router = APIRouter()

# Example route using the injected async session
@router.get("/users/")
async def list_users(session: DB_Session):
    # Perform database operations asynchronously
    result = await session.execute("SELECT * FROM users;")
    return result.fetchall()
```

## 2. Examples of Asynchronous Database Operations (CRUD)

When using `AsyncSession`, database operations must be prefixed with `await`.

### Creating Data (POST Example)

To create a new record, the ORM object is added to the session, and the transaction is committed asynchronously:

``` python
# Assuming Song is an ORM mapped class
@router.post("/songs")
async def add_song(song_data: SongCreate, session: AsyncSession = Depends(get_session)):
    song = Song(name=song_data.name, artist=song_data.artist)

    session.add(song)
    # Commit must be awaited
    await session.commit()

    # Refresh to load auto-generated fields (like ID)
    await session.refresh(song)

    return song
```

### Reading Data (GET Example)

Retrieving data involves executing a query statement (often using `select` imported from SQLAlchemy) and awaiting the result:

``` python
from sqlalchemy import select
from sqlmodel.ext.asyncio.session import AsyncSession # Or use AsyncSession from sqlalchemy.ext.asyncio
# Assuming Song is the model

@router.get("/songs", response_model=list[Song])
async def get_songs(session: AsyncSession = Depends(get_session)):
    stmt = select(Song)

    # Execution must be awaited
    result = await session.execute(stmt)

    # Accessing scalar results
    songs = result.scalars().all()

    return [Song(name=song.name, artist=song.artist, id=song.id) for song in songs]
```

### Updating Data (PUT Example)

Updating involves fetching the object, modifying it, and committing the changes:

``` python
from fastapi import HTTPException
from sqlalchemy import select

@router.put("/{todo_id}", summary="Update a todo")
async def update_todo(db: DB_Session, todo_id: int, request_data: UpdateTodoRequest) -> UpdateTodoResponse:
    stmt = select(models.Todo).where(
        models.Todo.id == todo_id,
        models.Todo.deleted_at.is_(None),
    )

    # Use .scalar() to retrieve the ORM object directly
    todo = (await db.execute(stmt)).scalar()

    if todo is None:
        raise HTTPException(status_code=404, detail="Todo not found")

    todo.content = request_data.content

    await db.commit()
    await db.refresh(todo)

    return UpdateTodoResponse(
        id=todo.id, content=todo.content, created_at=todo.created_at, updated_at=todo.updated_at
    )
```

## 3. Alembic for Asynchronous Database Migrations

Alembic is a database migration tool designed to manage changes to the database schema. It can be configured to work with Async SQLAlchemy.

### Initialization

To initialize Alembic for asynchronous support, the `-t`` ``async` template flag is used:

``` bash
alembic init -t async migrations
```

This command creates the necessary structure, including an `env.py` file optimized for asynchronous operations.

### Configuration for Asynchronous Execution

The `alembic/env.py` file must be configured to utilize Python’s `asyncio` and `async_engine_from_config` to run migrations in the “online” mode:

1.  **Set Database URL:** The configuration file (`alembic.ini`) or `env.py` must be updated to use the appropriate asynchronous database URL.

    ``` python
    # Inside alembic/env.py's run_migrations_online function setup:
    config.set_main_option("sqlalchemy.url", settings.SQLALCHEMY_DATABASE_URL)
    ```

2.  **Define Target Metadata:** The Alembic configuration needs to know which models (metadata) it should compare against the current database state for autogeneration:

    ``` python
    # Inside alembic/env.py
    from app.models.database import Base # Or SQLModel
    target_metadata = Base.metadata # or SQLModel.metadata
    ```

3.  **Run Migrations Asynchronously:** The provided async template modifies the `run_migrations_online` function to wrap synchronous operations (`do_run_migrations`) within an async execution context:

    ``` python
    import asyncio
    from sqlalchemy.ext.asyncio import async_engine_from_config

    # ... do_run_migrations function defined to handle context configuration synchronously ...

    async def run_async_migrations():
        # Creates the async engine
        connectable = async_engine_from_config(config.get_section(config.config_ini_section), ...)

        async with connectable.connect() as connection:
            # Runs synchronous DDL commands (like schema creation/alteration)
            await connection.run_sync(do_run_migrations)
        await connectable.dispose()

    def run_migrations_online():
        """Run migrations in 'online' mode."""
        asyncio.run(run_async_migrations())
    ```

### Generating and Applying Migrations

Once configured, migrations can be automatically generated by comparing the model metadata against the database:

``` bash
# Autogenerate a migration script
$ alembic revision --autogenerate -m "Initial tables"
```

The generated script, which uses Alembic’s `op` operations (e.g., `op.create_table`), is then applied:

``` bash
# Apply migrations up to the latest revision (head)
$ alembic upgrade head
```

Migration files typically contain synchronous SQLAlchemy operation calls within the `upgrade()` and `downgrade()` functions, which are executed safely within the asynchronous context managed by `env.py`. For example, manually adding and dropping a column:

``` python
def upgrade():
    op.add_column('user', sa.Column('new_column', sa.String()))

def downgrade():
    op.drop_column('user', 'new_column')
```

------------------------------------------------------------------------

## Modern Async Session Patterns (2026 Update)

### Use `async_sessionmaker` (Not `sessionmaker`)

SQLAlchemy 2.0+ provides a dedicated `async_sessionmaker` that is more type-safe and explicit than using `sessionmaker(class_=AsyncSession)`:

``` python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost:5432/mydb"

engine = create_async_engine(DATABASE_URL, echo=True)

# Recommended: use async_sessionmaker directly
async_session = async_sessionmaker(engine, expire_on_commit=False)


async def get_db():
    """FastAPI dependency: provides an async session per request."""
    async with async_session() as session:
        yield session
```

### Alembic 1.18 Bulk Reflection

Alembic 1.18 (January 2026) introduced **bulk reflection** for autogenerate — instead of querying the database once per table, it uses bulk inspector methods. This reduces autogenerate time from O(N) to O(1) for PostgreSQL and Oracle databases with many tables.
