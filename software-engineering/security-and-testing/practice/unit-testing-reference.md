# Pytest Reference Implementation

Complete reference test suite for a FastAPI application with authentication and CRUD operations.

## tests/conftest.py

``` python
import asyncio
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

from main import app
from db import get_db
from models import Base

# Test database (SQLite async for speed)
TEST_DATABASE_URL = "sqlite+aiosqlite:///./test.db"
test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
TestSession = sessionmaker(test_engine, class_=AsyncSession, expire_on_commit=False)


async def override_get_db():
    """Provide a test database session."""
    async with TestSession() as session:
        yield session


@pytest.fixture(scope="session")
def event_loop():
    """Create a single event loop for the entire test session."""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()


@pytest.fixture(autouse=True)
async def setup_db():
    """Create tables before each test, drop after."""
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest.fixture
async def client():
    """Async HTTP client for testing FastAPI endpoints."""
    app.dependency_overrides[get_db] = override_get_db
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()


@pytest.fixture
async def auth_headers(client: AsyncClient) -> dict:
    """Register a user and return Authorization headers."""
    await client.post("/auth/register", json={
        "name": "Test User",
        "email": "[email protected]",
        "password": "securepassword123",
    })
    response = await client.post("/auth/login", data={
        "username": "[email protected]",
        "password": "securepassword123",
    })
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}
```

## tests/test_auth.py

``` python
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
class TestRegistration:
    async def test_register_success(self, client: AsyncClient):
        response = await client.post("/auth/register", json={
            "name": "Alice",
            "email": "[email protected]",
            "password": "strongpassword",
        })
        assert response.status_code == 201
        data = response.json()
        assert data["email"] == "[email protected]"
        assert "hashed_password" not in data  # Must not leak password

    async def test_register_duplicate_email(self, client: AsyncClient):
        payload = {"name": "Alice", "email": "[email protected]", "password": "pass123"}
        await client.post("/auth/register", json=payload)
        response = await client.post("/auth/register", json=payload)
        assert response.status_code == 409

    async def test_register_invalid_email(self, client: AsyncClient):
        response = await client.post("/auth/register", json={
            "name": "Alice",
            "email": "not-an-email",
            "password": "pass123",
        })
        assert response.status_code == 422


@pytest.mark.asyncio
class TestLogin:
    async def test_login_success(self, client: AsyncClient):
        # Arrange: Register user first
        await client.post("/auth/register", json={
            "name": "Bob",
            "email": "[email protected]",
            "password": "bobpassword",
        })
        # Act: Login
        response = await client.post("/auth/login", data={
            "username": "[email protected]",
            "password": "bobpassword",
        })
        # Assert
        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert "refresh_token" in data
        assert data["token_type"] == "bearer"

    async def test_login_wrong_password(self, client: AsyncClient):
        await client.post("/auth/register", json={
            "name": "Carol",
            "email": "[email protected]",
            "password": "correctpass",
        })
        response = await client.post("/auth/login", data={
            "username": "[email protected]",
            "password": "wrongpass",
        })
        assert response.status_code == 401


@pytest.mark.asyncio
class TestProtectedEndpoints:
    async def test_access_without_token(self, client: AsyncClient):
        response = await client.get("/api/v1/users/me")
        assert response.status_code == 401

    async def test_access_with_valid_token(self, client: AsyncClient, auth_headers: dict):
        response = await client.get("/api/v1/users/me", headers=auth_headers)
        assert response.status_code == 200
        assert response.json()["email"] == "[email protected]"
```

## tests/test_tickets.py

``` python
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
class TestTicketCRUD:
    async def test_create_ticket(self, client: AsyncClient, auth_headers: dict):
        response = await client.post("/api/v1/tickets", json={
            "content": "Laptop not booting",
            "description": "Screen stays black after pressing power button",
        }, headers=auth_headers)
        assert response.status_code == 201
        data = response.json()
        assert data["content"] == "Laptop not booting"
        assert data["status"] == "pending"
        assert "ticket_id" in data

    async def test_create_ticket_without_auth(self, client: AsyncClient):
        response = await client.post("/api/v1/tickets", json={"content": "Test"})
        assert response.status_code == 401

    async def test_list_tickets(self, client: AsyncClient, auth_headers: dict):
        # Create two tickets
        await client.post("/api/v1/tickets", json={"content": "Ticket 1"}, headers=auth_headers)
        await client.post("/api/v1/tickets", json={"content": "Ticket 2"}, headers=auth_headers)

        response = await client.get("/api/v1/tickets", headers=auth_headers)
        assert response.status_code == 200
        data = response.json()
        assert len(data) == 2

    async def test_list_tickets_with_pagination(self, client: AsyncClient, auth_headers: dict):
        for i in range(5):
            await client.post("/api/v1/tickets", json={"content": f"Ticket {i}"}, headers=auth_headers)

        response = await client.get("/api/v1/tickets?skip=0&limit=2", headers=auth_headers)
        assert response.status_code == 200
        assert len(response.json()) == 2

    async def test_get_ticket_by_id(self, client: AsyncClient, auth_headers: dict):
        create_resp = await client.post("/api/v1/tickets", json={"content": "Find me"}, headers=auth_headers)
        ticket_id = create_resp.json()["ticket_id"]

        response = await client.get(f"/api/v1/tickets/{ticket_id}", headers=auth_headers)
        assert response.status_code == 200
        assert response.json()["content"] == "Find me"

    async def test_get_nonexistent_ticket(self, client: AsyncClient, auth_headers: dict):
        response = await client.get("/api/v1/tickets/nonexistent-uuid", headers=auth_headers)
        assert response.status_code == 404

    async def test_update_ticket_status(self, client: AsyncClient, auth_headers: dict):
        create_resp = await client.post("/api/v1/tickets", json={"content": "Pending"}, headers=auth_headers)
        ticket_id = create_resp.json()["ticket_id"]

        response = await client.put(f"/api/v1/tickets/{ticket_id}", json={
            "status": "in_progress",
        }, headers=auth_headers)
        assert response.status_code == 200
        assert response.json()["status"] == "in_progress"

    async def test_soft_delete_ticket(self, client: AsyncClient, auth_headers: dict):
        create_resp = await client.post("/api/v1/tickets", json={"content": "Delete me"}, headers=auth_headers)
        ticket_id = create_resp.json()["ticket_id"]

        response = await client.delete(f"/api/v1/tickets/{ticket_id}", headers=auth_headers)
        assert response.status_code == 204

        # Verify ticket is no longer in list
        list_resp = await client.get("/api/v1/tickets", headers=auth_headers)
        ids = [t["ticket_id"] for t in list_resp.json()]
        assert ticket_id not in ids
```

## Key Testing Patterns

| Pattern | Usage |
|----|----|
| **Fixture: `client`** | Creates async HTTP client with test DB override |
| **Fixture: `auth_headers`** | Registers + logs in a user, returns Bearer headers |
| **Fixture: `setup_db` (autouse)** | Fresh database for every test — full isolation |
| **`pytest.mark.asyncio`** | Required for async test functions |
| **Arrange-Act-Assert** | Each test follows the AAA pattern clearly |
| **Dependency override** | `app.dependency_overrides[get_db]` swaps real DB for test DB |
