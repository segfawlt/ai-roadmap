# Applied Assessment Quiz

This quiz covers all topics from the Applied tier. Answer each question, then check your answers.

------------------------------------------------------------------------

## Section 1: FastAPI Basics

**1. What makes FastAPI different from Flask? Select all that apply.**

- A\) Built-in async support (ASGI)

- B\) Automatic request/response validation via Pydantic

- C\) Auto-generated OpenAPI documentation

- D\) Requires more boilerplate code than Flask

Answer

**A, B, C** — FastAPI is built on ASGI (async-native), uses Pydantic for automatic validation, and auto-generates Swagger/ReDoc docs. It actually requires *less* boilerplate than Flask for equivalent functionality. D is incorrect.

**2. What is the role of Uvicorn in a FastAPI application?**

Answer

Uvicorn is an ASGI server that runs the FastAPI application. FastAPI defines the application logic; Uvicorn handles the actual HTTP connections, receives requests, passes them to FastAPI, and sends responses back. The relationship is: **Client → Uvicorn (ASGI server) → FastAPI (application)**.

------------------------------------------------------------------------

## Section 2: ASGI

**3. What is ASGI and how does it differ from WSGI?**

- A\) ASGI is synchronous; WSGI is asynchronous

- B\) ASGI supports async/await, WebSockets, and HTTP/2; WSGI is synchronous only

- C\) ASGI and WSGI are identical

- D\) ASGI only works with Django

Answer

**B** — ASGI (Asynchronous Server Gateway Interface) is the async evolution of WSGI. It supports async/await for non-blocking I/O, WebSocket connections, HTTP/2, and background tasks — all impossible with WSGI’s synchronous request-response model.

------------------------------------------------------------------------

## Section 3: Parameters

**4. What is the difference between path parameters, query parameters, and request body in FastAPI?**

Answer

| Type | Location | Use Case | Example |
|----|----|----|----|
| **Path** | URL path | Identify a specific resource | `GET`` ``/users/42` |
| **Query** | URL after `?` | Filter, sort, paginate | `GET`` ``/users?status=active&limit=10` |
| **Body** | Request body (JSON) | Send complex data for creation/update | `POST`` ``/users` with JSON payload |

In FastAPI:

``` python
@app.get("/users/{user_id}")  # Path parameter
async def get_user(
    user_id: int,              # From path
    include_orders: bool = False,  # Query parameter (has default)
):
    ...

@app.post("/users")
async def create_user(user: UserCreate):  # Request body (Pydantic model)
    ...
```

**5. How does FastAPI automatically validate a path parameter like `user_id:`` ``int`?**

Answer

FastAPI uses Python type hints. When you declare `user_id:`` ``int`, FastAPI automatically:

1.  Extracts the value from the URL path

2.  Attempts to convert it to `int`

3.  Returns a `422`` ``Unprocessable`` ``Entity` with a clear error message if conversion fails (e.g., `/users/abc`)

4.  Passes the validated integer to your function

------------------------------------------------------------------------

## Section 4: Pydantic Models

**6. What is the purpose of `response_model` in FastAPI?**

- A\) It validates the request body

- B\) It filters the response to only include specified fields (preventing data leaks)

- C\) It creates the database table

- D\) It generates the API documentation only

Answer

**B** — `response_model` controls what fields are included in the API response. Even if your database model has `hashed_password`, setting `response_model=UserResponse` (which excludes password) ensures sensitive data is never sent to the client. It also validates the response shape and generates accurate OpenAPI docs.

**7. Why should you use separate Pydantic models for Create, Update, and Response?**

Answer

Different operations need different fields:

- **Create** (`UserCreate`): Requires `password`, but not `id` or `created_at` (auto-generated)

- **Update** (`UserUpdate`): All fields optional (partial update)

- **Response** (`UserResponse`): Includes `id` and `created_at`, excludes `hashed_password`

Using a single model would either expose sensitive data or require fields that shouldn’t be required.

------------------------------------------------------------------------

## Section 5: Database Setup

**8. In SQLAlchemy 2.0, what is the difference between `Session` and `AsyncSession`?**

Answer

`Session` is synchronous — it blocks the thread during database operations. `AsyncSession` is async — it uses `await` and does not block the event loop. For FastAPI (which is async-native), `AsyncSession` is preferred because it allows the server to handle other requests while waiting for database queries.

``` python
# Sync
with Session(engine) as session:
    user = session.query(User).first()

# Async
async with AsyncSession(engine) as session:
    result = await session.execute(select(User))
    user = result.scalars().first()
```

**9. What is the purpose of `Depends(get_db)` in FastAPI?**

Answer

`Depends(get_db)` is dependency injection. It:

1.  Calls `get_db()` before the endpoint runs (creates a database session)

2.  Passes the session to the endpoint function

3.  Cleans up the session after the endpoint returns (via `yield`)

This ensures every request gets a fresh session and the session is always properly closed, even if an error occurs.

------------------------------------------------------------------------

## Section 6: Alembic Migrations

**10. What problem does Alembic solve?**

- A\) It replaces SQLAlchemy for database queries

- B\) It tracks and applies database schema changes (migrations) in a versioned, reproducible way

- C\) It automatically creates REST endpoints

- D\) It provides database connection pooling

Answer

**B** — Alembic manages database schema evolution. When you add a column, create a table, or change a type, Alembic generates a migration script that can be applied to any database (development, staging, production) to bring it to the same state. This is essential for team development and deployment.

**11. What is the difference between `alembic`` ``revision`` ``--autogenerate` and writing migrations manually?**

Answer

`--autogenerate` compares your SQLAlchemy models with the current database schema and generates a migration script for the differences. However, it cannot detect:

- Table or column renames (it sees a drop + create instead)

- Data migrations (moving data between columns)

- Custom constraints or triggers

For these cases, you must write migrations manually. Best practice: always review auto-generated migrations before applying them.

**12. You need to add a `phone` column to the `users` table in production. Describe the steps.**

Answer

1.  Add the field to your SQLAlchemy model: `phone`` ``=`` ``Column(String,`` ``nullable=True)`

2.  Generate migration: `alembic`` ``revision`` ``--autogenerate`` ``-m`` ``"add`` ``phone`` ``column`` ``to`` ``users"`

3.  Review the generated migration script in `alembic/versions/`

4.  Test locally: `alembic`` ``upgrade`` ``head`

5.  Commit the migration script to Git

6.  In production: `alembic`` ``upgrade`` ``head` (run before deploying new application code)

Key: Use `nullable=True` for the new column so existing rows are not affected. If you need a default value, set it in the migration.
