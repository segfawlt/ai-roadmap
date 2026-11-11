# Exercise: Migrations (Alembic), Concurrency & Transactions

- Refer to `U03-01.TodoAPI` in Unit 03

## Goal

Use **Alembic** for schema management and add **transaction-safe operations** and **indexes**.

> **Note:** Alembic typically runs with a **sync driver** (e.g., `psycopg`/`psycopg2`) even if the app uses `asyncpg`. Keep migrations separate from runtime DB access.

## Tasks

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

## Acceptance Criteria

- Alembic revision adds new columns/indexes.

- Toggle endpoint guarantees consistency under concurrent requests (explain your approach).

## Deliverables

- Alembic migration files.

- Brief concurrency note in README.

## Rubric

- Alembic set up & run

- Correct schema changes & indexes

- Concurrency-safe operation
