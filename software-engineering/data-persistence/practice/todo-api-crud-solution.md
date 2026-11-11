# Exercise: End-to-End Solution

1.  **Exercise 1 — Environment & Postgres on Docker CE**

2.  **Exercise 2 — FastAPI + Async SQLAlchemy + CRUD for TODOs**

3.  **Exercise 3 — Pagination, Filtering, Validation, and Error Handling**

4.  **Exercise 3 — Database Migrations (Alembic), Concurrency & Transactions**

> **Target duration:** Each lab 60–120 minutes. Capstone 2–4 hours. **Prereqs:** Python 3.10+, Linux with Docker CE installed (no Desktop), basic SQL, REST.

------------------------------------------------------------------------

## Lab 1 — Setup PostgreSQL on Docker **CE** (not Desktop)

### Goal

Run a Postgres instance in **Docker Engine - Community** and verify connectivity.

### Tasks

1.  **Install Docker CE (Engine) on Linux** (Ubuntu/Debian recommended).

    - Verify with:

      ``` bash
      docker --version
      docker info | grep -E "Server|Engine|Community"
      docker compose version   # plugin v2 preferred
      ```

    - **Acceptance proof:** `docker`` ``info` shows `Server:`` ``Docker`` ``Engine`` ``-`` ``Community`. **Any evidence of “Docker Desktop” = fail.**

2.  **Start PostgreSQL via Docker Compose (or docker run)**

    - Compose file (`compose.yaml`):

      ``` yaml
      services:
        db:
          image: postgres:16-alpine
          container_name: todo_db
          environment:
            POSTGRES_USER: todo_user
            POSTGRES_PASSWORD: todo_pass
            POSTGRES_DB: todo_db
          ports:
            - "5432:5432"
          volumes:
            - pg_data:/var/lib/postgresql/data
          healthcheck:
            test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
            interval: 5s
            timeout: 3s
            retries: 10
      volumes:
        pg_data:
          driver: local
      ```

    - If compose plugin isn’t available, use:

      ``` bash
      docker network create todo_net
      docker run -d --name todo_db \
        --net todo_net \
        -e POSTGRES_USER=todo_user \
        -e POSTGRES_PASSWORD=todo_pass \
        -e POSTGRES_DB=todo_db \
        -p 5432:5432 \
        -v pg_data:/var/lib/postgresql/data \
        postgres:16-alpine
      ```

3.  **Verify database connectivity**

    - From host or a psql client container:

      ``` bash
      docker exec -it todo_db psql -U todo_user -d todo_db -c "\dt"
      ```

### Acceptance Criteria

- Evidence `docker`` ``info` shows **Community** (CE).

- Postgres container healthy (`docker`` ``ps`, healthcheck passing).

- Can run `SELECT`` ``1` against the DB.

### Deliverables

- Screenshot/log of `docker`` ``info` indicating CE, container status, and a successful query.

### Rubric (10 pts)

- Docker CE installed & verified (3)

- Postgres container running & healthy (4)

- Connectivity verified (3)

------------------------------------------------------------------------

## Lab 2 — FastAPI + Async SQLAlchemy + Basic TODO CRUD

### Goal

Build an API-only TODO service using **FastAPI** + **SQLAlchemy 2.x async** (`asyncpg`) connected to the Dockerized Postgres.

### Suggested Project Structure

```
app/
  core/config.py
  db/session.py
  db/models.py
  db/repository.py
  api/routes/todos.py
  main.py
alembic/  (created in Lab 4)
compose.yaml (later adds app service)
Dockerfile
```

### Tasks

1.  **Dependencies**

    ``` bash
    pip install fastapi uvicorn sqlalchemy asyncpg pydantic pydantic-settings
    ```

2.  **Async SQLAlchemy setup**

    ``` python
    # app/db/session.py
    from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
    from sqlalchemy.orm import DeclarativeBase

    class Base(DeclarativeBase): pass

    DATABASE_URL = "postgresql+asyncpg://todo_user:todo_pass@localhost:5432/todo_db"

    engine = create_async_engine(DATABASE_URL, echo=False, pool_size=5, max_overflow=10)
    async_session = async_sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)
    ```

3.  **Model**

    ``` python
    # app/db/models.py
    from sqlalchemy import String, Boolean, DateTime, func, Integer
    from sqlalchemy.orm import Mapped, mapped_column
    from .session import Base

    class Todo(Base):
      __tablename__ = "todos"
      id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
      title: Mapped[str] = mapped_column(String(200))
      description: Mapped[str | None] = mapped_column(String(1000), nullable=True)
      completed: Mapped[bool] = mapped_column(Boolean, default=False)
      created_at: Mapped[DateTime] = mapped_column(DateTime, server_default=func.now())
      updated_at: Mapped[DateTime] = mapped_column(DateTime, onupdate=func.now(), nullable=True)
    ```

4.  **Schema (Pydantic) & Routes**

    ``` python
    # app/api/routes/todos.py
    from fastapi import APIRouter, Depends, HTTPException, status
    from pydantic import BaseModel, Field
    from sqlalchemy.ext.asyncio import AsyncSession
    from sqlalchemy import select
    from app.db.session import async_session
    from app.db.models import Todo

    router = APIRouter(prefix="/todos", tags=["todos"])

    class TodoCreate(BaseModel):
      title: str = Field(min_length=1, max_length=200)
      description: str | None = Field(default=None, max_length=1000)

    class TodoRead(BaseModel):
      id: int
      title: str
      description: str | None
      completed: bool

      class Config:
        from_attributes = True  # SQLAlchemy 2.x

    async def get_session() -> AsyncSession:
      async with async_session() as session:
        yield session

    @router.post("", response_model=TodoRead, status_code=status.HTTP_201_CREATED)
    async def create_todo(payload: TodoCreate, session: AsyncSession = Depends(get_session)):
      todo = Todo(title=payload.title, description=payload.description)
      session.add(todo)
      await session.commit()
      await session.refresh(todo)
      return todo

    @router.get("/{todo_id}", response_model=TodoRead)
    async def get_one(todo_id: int, session: AsyncSession = Depends(get_session)):
      todo = await session.get(Todo, todo_id)
      if not todo:
        raise HTTPException(status_code=404, detail="Todo not found")
      return todo

    @router.patch("/{todo_id}", response_model=TodoRead)
    async def toggle_complete(todo_id: int, session: AsyncSession = Depends(get_session)):
      todo = await session.get(Todo, todo_id)
      if not todo:
        raise HTTPException(status_code=404, detail="Todo not found")
      todo.completed = not todo.completed
      await session.commit()
      await session.refresh(todo)
      return todo

    @router.delete("/{todo_id}", status_code=status.HTTP_204_NO_CONTENT)
    async def delete_one(todo_id: int, session: AsyncSession = Depends(get_session)):
      todo = await session.get(Todo, todo_id)
      if not todo:
        return  # 204 is fine for idempotent deletes
      await session.delete(todo)
      await session.commit()
      return
    ```

5.  **FastAPI App**

    ``` python
    # app/main.py
    from fastapi import FastAPI
    from app.api.routes.todos import router as todo_router
    from app.db.session import engine, Base

    app = FastAPI(title="TODO API")

    @app.on_event("startup")
    async def on_startup():
      # For lab 2 only: create tables programmatically (later use Alembic)
      async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    app.include_router(todo_router)
    ```

6.  **Run**

    ``` bash
    uvicorn app.main:app --reload
    # Test in /docs (Swagger) and with curl
    curl -X POST http://localhost:8000/todos -H "Content-Type: application/json" -d '{"title":"Read paper"}'
    ```

### Acceptance Criteria

- API responds with 201 on create, returns JSON aligned with schema.

- GET, PATCH toggle, DELETE behave as defined.

- `/docs` loads correctly with generated OpenAPI.

### Deliverables

- Source code.

- A short README with run instructions and sample curl calls.

### Rubric (20 pts)

- Async DB setup (5)

- Model + schema correctness (5)

- CRUD endpoints working (8)

- README & API docs (2)

------------------------------------------------------------------------

## Lab 4 — Migrations (Alembic), Concurrency & Transactions

### Goal

Use **Alembic** for schema management and add **transaction-safe operations** and **indexes**.

> **Note:** Alembic typically runs with a **sync driver** (e.g., `psycopg`/`psycopg2`) even if the app uses `asyncpg`. Keep migrations separate from runtime DB access.

### Tasks

1.  **Add fields**: `due_date` (nullable), `priority` (enum or int).

2.  **Initialize Alembic**

    ``` bash
    pip install alembic psycopg[binary]  # psycopg3
    alembic init alembic
    ```

    - Set `sqlalchemy.url` in `alembic.ini` to sync URL: postgresql+psycopg://todo_user:todo_pass@localhost:5432/todo_db

3.  **Create revision & upgrade**

    - Autogenerate revision or handwrite `upgrade()`/`downgrade()`.

    - Run:

      ``` bash
      alembic upgrade head
      ```

4.  **Indexes**

    - Index on `completed`, `created_at`.

5.  **Transaction-safe toggle**

    - Use `SELECT`` ``...`` ``FOR`` ``UPDATE` or session transactions to avoid race conditions when toggling `completed` in high concurrency.

### Acceptance Criteria

- Alembic revision adds new columns/indexes.

- Toggle endpoint guarantees consistency under concurrent requests (explain your approach).

### Deliverables

- Alembic migration files.

- Brief concurrency note in README.

### Rubric (25 pts)

- Alembic set up & run (10)

- Correct schema changes & indexes (8)

- Concurrency-safe operation (7)
