# CRUD Application Overview

This page describes the architecture of a CRUD API built with FastAPI, SQLAlchemy, Pydantic, and Alembic, showing how routes, database sessions, ORM models, and schemas fit together in a clean, layered structure.

## Big picture

``` mermaid
        graph TD
    Client[HTTP Client] --> Routes[FastAPI Routes<br>main.py]
    Routes -->|Depends get_db| DI[Dependency Injection<br>database.py]
    Routes -->|validate input| Schemas[Pydantic Schemas<br>schemas.py]
    Routes -->|call CRUD functions| CRUD[CRUD Layer<br>crud.py]
    DI -->|Session| CRUD
    CRUD -->|ORM queries| Models[SQLAlchemy Models<br>models.py]
    Models -->|SQL| DB[(Database)]
    Alembic[Alembic Migrations<br>db-migration/] -->|manage schema| DB
    style Routes fill:#9cf,stroke:#333
    style CRUD fill:#9f9,stroke:#333
    style Models fill:#fc9,stroke:#333
    style DB fill:#f9c,stroke:#333
```

A CRUD (Create, Read, Update, Delete) FastAPI application typically consists of:

- **FastAPI** app with route handlers (path operation functions)

- **SQLAlchemy** models and session management as dependencies

- **Pydantic** schemas for request/response validation

- **Alembic** for database migrations

- **Dependency injection** to share database sessions across routes

The pattern separates concerns cleanly: routes handle HTTP, dependencies manage resources, and models handle data persistence.

## Typical Project Structure

```
app/
├── main.py          # FastAPI app, route registration
├── database.py      # Engine, session factory, get_db dependency
├── models.py        # SQLAlchemy ORM models
├── schemas.py       # Pydantic request/response schemas
├── crud.py          # Database query functions
└── db-migration/    # Alembic migration files
```

## Example: CRUD Endpoints

``` python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session

from . import crud, models, schemas
from .database import engine, get_db

models.Base.metadata.create_all(bind=engine)

app = FastAPI()

@app.post("/items/", response_model=schemas.Item)
def create_item(item: schemas.ItemCreate, db: Session = Depends(get_db)):
    return crud.create_item(db=db, item=item)

@app.get("/items/", response_model=list[schemas.Item])
def read_items(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    items = crud.get_items(db, skip=skip, limit=limit)
    return items

@app.get("/items/{item_id}", response_model=schemas.Item)
def read_item(item_id: int, db: Session = Depends(get_db)):
    db_item = crud.get_item(db, item_id=item_id)
    if db_item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return db_item

@app.put("/items/{item_id}", response_model=schemas.Item)
def update_item(item_id: int, item: schemas.ItemCreate, db: Session = Depends(get_db)):
    db_item = crud.update_item(db, item_id=item_id, item=item)
    if db_item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return db_item

@app.delete("/items/{item_id}")
def delete_item(item_id: int, db: Session = Depends(get_db)):
    success = crud.delete_item(db, item_id=item_id)
    if not success:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"detail": "deleted"}
```

## The `get_db` Dependency Pattern

``` python
from sqlalchemy.orm import Session
from .database import SessionLocal

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

FastAPI sees `db=Depends(get_db)`, calls `get_db()`, injects the yielded session, and handles cleanup after the request completes.

## Response Model Filtering

Using `response_model` on a route automatically filters the response to only include fields defined in the schema:

``` python
class UserOut(BaseModel):
    id: int
    username: str
    # password is NOT included — even if the ORM model has it

@app.get("/users/{user_id}", response_model=UserOut)
def get_user(user_id: int, db: Session = Depends(get_db)):
    return crud.get_user(db, user_id)
```

This is a security feature: it prevents accidentally exposing sensitive fields even if they exist in the database model.
