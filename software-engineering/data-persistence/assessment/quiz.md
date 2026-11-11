# Integration Assessment Quiz

This quiz covers all topics from the Integration tier. Answer each question, then check your answers.

------------------------------------------------------------------------

## Section 1: Async Database

**1. Why is `AsyncSession` preferred over `Session` in FastAPI applications?**

Answer

FastAPI is an async framework running on an ASGI server. Using synchronous `Session` blocks the event loop during database queries, preventing the server from handling other requests. `AsyncSession` uses `await`, allowing the event loop to process other requests while waiting for the database — resulting in much higher concurrency under load.

**2. What is the correct pattern for providing an async database session in FastAPI?**

``` python
# Which is correct: A or B?

# A
async def get_db():
    session = AsyncSession(engine)
    try:
        yield session
    finally:
        await session.close()

# B
async def get_db():
    session = AsyncSession(engine)
    return session
```

Answer

**A** is correct. Using `yield` makes it a dependency generator:

- FastAPI calls it, gets the session (before `yield`)

- Passes the session to the endpoint

- After the endpoint returns (or raises), the `finally` block runs and closes the session

**B** is wrong because the session is never closed — it would leak database connections.

**3. What happens if you forget `await` before a database query in an async endpoint?**

Answer

The query is never executed. You get a coroutine object instead of the actual result. For example:

``` python
# BUG: Missing await
result = session.execute(select(User))  # Returns a coroutine, not results
users = result.scalars().all()  # AttributeError: 'coroutine' has no attribute 'scalars'

# CORRECT
result = await session.execute(select(User))
users = result.scalars().all()
```

------------------------------------------------------------------------

## Section 2: Dependency Injection

**4. Explain how FastAPI’s `Depends()` system works. What are its benefits?**

Answer

`Depends()` declares that an endpoint requires a dependency. FastAPI:

1.  Calls the dependency function before the endpoint

2.  Passes the result to the endpoint parameter

3.  Handles cleanup (for `yield` dependencies)

4.  Caches dependencies within a single request (if the same dependency is used multiple times)

**Benefits:**

- **Separation of concerns**: Auth, DB sessions, and config are separate from business logic

- **Testability**: Override dependencies in tests with mocks/fakes

- **Composability**: Dependencies can depend on other dependencies (dependency graph)

- **Reusability**: Define once, use across many endpoints

**5. How would you create a dependency that requires authentication?**

``` python
# Fill in the implementation
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    # ???
```

Answer

``` python
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

    result = await db.execute(select(User).where(User.id == int(user_id)))
    user = result.scalars().first()
    if user is None:
        raise HTTPException(status_code=401, detail="User not found")
    return user
```

Then use it in endpoints: `current_user:`` ``User`` ``=`` ``Depends(get_current_user)`

**6. How do you override a dependency in tests?**

Answer

Use `app.dependency_overrides`:

``` python
# In tests/conftest.py
from main import app, get_db

async def override_get_db():
    async with AsyncSession(test_engine) as session:
        yield session

app.dependency_overrides[get_db] = override_get_db
```

This replaces the real database session with a test database session, without changing any application code.

------------------------------------------------------------------------

## Section 3: CRUD Operations

**7. Design a CRUD service for a `Ticket` model. What methods should it have?**

Answer

``` python
class TicketService:
    async def create(self, db: AsyncSession, data: TicketCreate, user_id: int) -> Ticket
    async def get_by_id(self, db: AsyncSession, ticket_id: str) -> Ticket | None
    async def list_by_user(self, db: AsyncSession, user_id: int, skip: int, limit: int) -> list[Ticket]
    async def update_status(self, db: AsyncSession, ticket_id: str, status: TicketStatus) -> Ticket
    async def soft_delete(self, db: AsyncSession, ticket_id: str) -> None
```

Key design decisions:

- Accept `db` session as parameter (injected by FastAPI)

- Separate `user_id` from the data payload (comes from auth, not user input)

- Use soft delete (status change) instead of hard delete

- Return domain objects, not dictionaries

**8. What is the difference between hard delete and soft delete? When would you use each?**

Answer

| Approach | What Happens | When to Use |
|----|----|----|
| **Hard delete** | Row is permanently removed from database | Temporary data, GDPR compliance (right to erasure) |
| **Soft delete** | Row is marked with `deleted_at` timestamp, filtered from normal queries | Audit trails, recoverable data, referential integrity |

Soft delete is preferred for most business entities (tickets, bookings, users) because:

- Data can be recovered if deleted by mistake

- Foreign key relationships remain intact

- Audit trail is preserved

**9. A junior developer writes this endpoint. What are the problems?**

``` python
@router.get("/tickets/{ticket_id}")
async def get_ticket(ticket_id: str, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Ticket).where(Ticket.ticket_id == ticket_id))
    ticket = result.scalars().first()
    return ticket
```

Answer

Three problems:

1.  **No 404 handling**: Returns `null`/`None` instead of `404`` ``Not`` ``Found` when ticket doesn’t exist

2.  **No authorization check**: Any authenticated user can view any ticket — should verify `ticket.user_id`` ``==`` ``current_user.id`

3.  **No response_model**: Returns the raw SQLAlchemy model which may include sensitive fields

Fixed:

``` python
@router.get("/tickets/{ticket_id}", response_model=TicketResponse)
async def get_ticket(
    ticket_id: str,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    result = await db.execute(select(Ticket).where(Ticket.ticket_id == ticket_id))
    ticket = result.scalars().first()
    if ticket is None:
        raise HTTPException(status_code=404, detail="Ticket not found")
    if ticket.user_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not authorized")
    return ticket
```

**10. How do you implement pagination in FastAPI with SQLAlchemy?**

Answer

``` python
@router.get("/tickets", response_model=list[TicketResponse])
async def list_tickets(
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=20, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    stmt = (
        select(Ticket)
        .where(Ticket.user_id == current_user.id)
        .order_by(Ticket.created_at.desc())
        .offset(skip)
        .limit(limit)
    )
    result = await db.execute(stmt)
    return result.scalars().all()
```

Key points:

- Validate `skip` (\>= 0) and `limit` (1-100) with `Query()`

- Always include `ORDER`` ``BY` for deterministic pagination

- Filter by `user_id` so users only see their own data

- Cap `limit` to prevent clients from requesting millions of rows
