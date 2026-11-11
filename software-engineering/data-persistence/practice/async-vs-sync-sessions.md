# 1. **Core Concept**

**Synchronous SQLAlchemy (classic)**:

- Operations block the Python thread until they complete.

- Typical usage:

``` python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.orm import sessionmaker, declarative_base

Base = declarative_base()
engine = create_engine("sqlite:///test.db")
Session = sessionmaker(bind=engine)

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)

# Sync usage
with Session() as session:
    user = session.query(User).filter_by(name="Alice").first()
    print(user.id)
```

**Asynchronous SQLAlchemy (`asyncio`)**:

- Uses `async`/`await`, so your code doesn’t block the event loop while waiting for the database.

- Requires an **async-compatible database driver**, like `asyncpg` for PostgreSQL or `aiosqlite` for SQLite.

- Typical usage:

``` python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker, declarative_base
import asyncio

Base = declarative_base()
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_user():
    async with AsyncSessionLocal() as session:
        result = await session.execute(
            "SELECT * FROM users WHERE name=:name", {"name": "Alice"}
        )
        user = result.fetchone()
        print(user)

asyncio.run(get_user())
```

------------------------------------------------------------------------

## 2. **Key Differences**

| Aspect | Synchronous | Asynchronous |
|----|----|----|
| **Execution** | Blocks thread until query finishes | Non-blocking, allows other coroutines to run while waiting |
| **Driver** | Standard DB drivers (psycopg2, pymysql, sqlite3) | Async drivers (asyncpg, aiosqlite, aiomysql) |
| **Session** | `Session` | `AsyncSession` |
| **Query Execution** | Direct function calls: `session.query().all()` | `await`` ``session.execute()` |
| **Engine** | `create_engine()` | `create_async_engine()` |
| **Transactions** | `with`` ``session.begin():`` ``...` | `async`` ``with`` ``session.begin():`` ``...` |
| **Performance** | Good for single-threaded or blocking apps | Better for high-concurrency async apps (web servers, async tasks) |
| **ORM Support** | Full ORM | Full ORM, but you must `await` query results |

------------------------------------------------------------------------

## 3. **Important Notes**

1.  **Async is not faster for a single query**: It mainly helps if your app is doing **many concurrent DB operations**, like a web server handling hundreds of requests simultaneously.

2.  **Mixing sync and async sessions is tricky**: You should generally stick to one pattern in a given context.

3.  **Some SQLAlchemy features are slightly different in async mode**:

    - `session.execute()` returns a `Result` object; you usually use `.fetchone()`, `.fetchall()`, or `.scalars()`.

    - Lazy loading of relationships is async too (`await`` ``user.addresses`).
