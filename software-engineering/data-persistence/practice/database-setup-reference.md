# Database Setup Reference

Complete reference implementation for SQLAlchemy async database setup with FastAPI.

## db.py — Database Connection

``` python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker, DeclarativeBase

# Connection URL — load from environment in production
DATABASE_URL = "postgresql+asyncpg://todo_user:todo_pass@localhost:5432/todo_db"

# Engine: manages the connection pool
engine = create_async_engine(
    DATABASE_URL,
    echo=True,       # Log SQL queries (disable in production)
    pool_size=5,     # Max persistent connections
    max_overflow=10, # Extra connections under load
)

# Session factory: creates new sessions for each request
async_session = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,  # Keep objects usable after commit
)


class Base(DeclarativeBase):
    """Base class for all SQLAlchemy models."""
    pass


async def get_db():
    """FastAPI dependency: provides a database session per request.

    Uses 'yield' so the session is properly closed after the
    request finishes, even if an error occurs.
    """
    async with async_session() as session:
        yield session
```

## models.py — Model Definition

``` python
from datetime import datetime
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey, Text
from sqlalchemy.orm import relationship

from db import Base


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), nullable=False)
    email = Column(String(255), unique=True, nullable=False, index=True)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)

    # Relationships
    todos = relationship("Todo", back_populates="user")


class Todo(Base):
    __tablename__ = "todos"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    description = Column(Text, nullable=True)
    completed = Column(Boolean, default=False)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    user = relationship("User", back_populates="todos")
```

## schemas.py — Pydantic Models

``` python
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime


# --- User schemas ---

class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    password: str = Field(min_length=8)


class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    is_active: bool
    created_at: datetime

    model_config = {"from_attributes": True}


# --- Todo schemas ---

class TodoCreate(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    description: str | None = None


class TodoResponse(BaseModel):
    id: int
    title: str
    description: str | None
    completed: bool
    created_at: datetime
    updated_at: datetime

    model_config = {"from_attributes": True}
```

## main.py — Application Setup

``` python
from contextlib import asynccontextmanager
from fastapi import FastAPI

from db import engine, Base


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Create tables on startup (dev only — use Alembic in production)."""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield


app = FastAPI(title="Todo API", lifespan=lifespan)

# Import and include routers
from routers.todos import router as todo_router
app.include_router(todo_router)
```

## docker-compose.yml — PostgreSQL Setup

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
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U todo_user -d todo_db"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

## Key Design Decisions

| Decision | Rationale |
|----|----|
| `asyncpg` driver | Native async PostgreSQL driver — no GIL blocking during queries |
| `expire_on_commit=False` | Allows accessing model attributes after `commit()` without extra queries |
| `yield` in `get_db()` | Ensures session is closed even if the endpoint raises an exception |
| Separate `schemas.py` | Pydantic models (API contract) are independent from SQLAlchemy models (DB schema) |
| `from_attributes`` ``=`` ``True` | Allows Pydantic to read data from SQLAlchemy model attributes directly |
| `pool_size=5` | Default connection pool — tune based on expected concurrent requests |
