# Exercise: FastAPI + Async SQLAlchemy + Basic TODO CRUD

## Goal

Build an API-only TODO service using **FastAPI** + **SQLAlchemy 2.x async** (`asyncpg`) connected to the Dockerized Postgres.

## Suggested Project Structure

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

## Tasks

1.  **Dependencies**

    ``` bash
    pip install fastapi uvicorn sqlalchemy asyncpg pydantic pydantic-settings
    ```

2.  **Async SQLAlchemy setup**

    - Depend on your implementation

3.  **Model**

    - `Todo` model has some fields: `id,`` ``title,`` ``description,`` ``completed,`` ``created_at,`` ``updated_at`

4.  **Schema (Pydantic)**

    - Create schema models: `TodoCreate`, `TodoRead` with require validation

5.  **FastAPI App**

    - Consider to use `router` (APIRouter - fastapi)

6.  **Run**

    ``` bash
    uvicorn app.main:app --reload
    # Test in /docs (Swagger) and with curl
    curl -X POST http://localhost:8000/todos -H "Content-Type: application/json" -d '{"title":"Read paper"}'
    ```

## Acceptance Criteria

- API responds with 201 on create, returns JSON aligned with schema.

- GET, PATCH toggle, DELETE behave as defined.

- `/docs` loads correctly with generated OpenAPI.

## Deliverables

- Source code.

- A short README with run instructions and sample curl calls.

## Rubric (40 pts)

- Async DB setup

- Model + schema correctness

- CRUD endpoints working

- README & API docs
