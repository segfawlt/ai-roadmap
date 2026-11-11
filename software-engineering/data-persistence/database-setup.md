# Install required packages

``` sh
# Install SQLAlchemy: for ORM
pip install SQLAlchemy
# Optional asyncio
pip install "sqlalchemy[asyncio]"


# Install asyncpg: for async operations
pip install asyncpg


# Install Pydantic: for validation
pip install pydantic
# For email validation
pip install "pydantic[email]"



# Install fastAPI
pip install fastapi uvicorn
```

## Follow the code structure

app/ ├── main.py ├── db.py ├── models.py ├── schemas.py └── crud.py

## Run to init model (database)

``` sh
python db.py
```

``` python
async def init_models():
    async with engine.begin() as conn:
        # Run the synchronous `create_all` within the async context
        await conn.run_sync(Base.metadata.create_all)

import asyncio


# Run the async function
if __name__ == "__main__":
    asyncio.run(init_models())
```

## SQLAlchemy 2.1 Updates (2026)

SQLAlchemy 2.1 (beta released January 2026) brings significant improvements:

### Key Changes

| Feature | What Changed |
|----|----|
| **Improved typing** | `Row` type returns individual column types without `Tuple` wrapper |
| **`filter_by()` scope** | Now searches across all FROM clause entities, not just the primary |
| **CREATE VIEW** | Native DDL support for database views |
| **Server-side UUIDv7** | Monotonic UUID generation in batched INSERT operations |
| **Free-threading** | Accommodations for Python 3.14 free-threaded builds |
| **`psycopg` (v3)** | Improved PostgreSQL dialect for the modern async driver |

### SQLModel: FastAPI-Native Alternative

For new API-first projects, consider [SQLModel](https://sqlmodel.tiangolo.com/) — it unifies Pydantic and SQLAlchemy models into a single class:

``` python
from sqlmodel import SQLModel, Field

# One class serves as both the database model AND the Pydantic schema
class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(index=True)
    email: str = Field(unique=True)
    age: int | None = None
```

**When to use SQLModel vs SQLAlchemy:**

| Scenario | Recommendation |
|----|----|
| New FastAPI project, simple models | **SQLModel** — less boilerplate, unified models |
| Complex relationships, legacy DB | **SQLAlchemy** — full control, mature ecosystem |
| Need raw SQL or advanced ORM features | **SQLAlchemy** — more flexible |

------------------------------------------------------------------------

## References

- <https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html#asyncio-platform-installation-notes-including-apple-m1>

- [SQLAlchemy 2.1 Migration Guide](https://www.sqlalchemy.org/docs/21/changelog/migration_21.html)

- [SQLModel Documentation](https://sqlmodel.tiangolo.com/)
