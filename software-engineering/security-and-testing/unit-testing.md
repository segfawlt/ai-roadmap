# Unit Testing FastAPI Applications

This page shows how to write unit tests for FastAPI applications using pytest, covering synchronous and asynchronous test clients, database mocking, and testing JWT-protected endpoints.

## Minimal FastAPI App to Test (`main.py`)

``` python
from fastapi import FastAPI, HTTPException

app = FastAPI()

fake_db = {"alice": "Engineer", "bob": "Designer"}

@app.get("/users/{username}")
def get_user(username: str):
    if username not in fake_db:
        raise HTTPException(status_code=404, detail="User not found")
    return {"username": username, "job": fake_db[username]}

@app.post("/users")
def create_user(username: str, job: str):
    if username in fake_db:
        raise HTTPException(status_code=400, detail="User already exists")
    fake_db[username] = job
    return {"message": "User created", "username": username}
```

## Sync Tests with `TestClient`

Requires:

``` bash
pip install pytest pytest-asyncio httpx fastapi
```

``` python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)


def test_get_existing_user():
    response = client.get("/users/alice")
    assert response.status_code == 200
    data = response.json()
    assert data["username"] == "alice"
    assert data["job"] == "Engineer"


def test_get_missing_user():
    response = client.get("/users/unknown")
    assert response.status_code == 404
    assert response.json() == {"detail": "User not found"}


def test_create_new_user():
    response = client.post("/users", params={"username": "charlie", "job": "Manager"})
    assert response.status_code == 200
    assert response.json()["message"] == "User created"


def test_create_existing_user():
    response = client.post("/users", params={"username": "alice", "job": "Engineer"})
    assert response.status_code == 400
    assert response.json() == {"detail": "User already exists"}
```

Run with:

``` bash
pytest -v
```

## Async Tests with `httpx.AsyncClient`

For async FastAPI endpoints, use `httpx.AsyncClient`:

``` python
import pytest
from httpx import AsyncClient
from main import app


@pytest.mark.asyncio
async def test_get_existing_user():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/users/alice")

    assert response.status_code == 200
    data = response.json()
    assert data["username"] == "alice"


@pytest.mark.asyncio
async def test_create_new_user():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(
            "/users", params={"username": "charlie", "job": "Manager"}
        )

    assert response.status_code == 200
    assert response.json()["message"] == "User created"
```

## Async Tests with Mocked Database

For production apps using SQLAlchemy async sessions, override the dependency with an in-memory test database:

### Setup (`database.py`)

``` python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker, declarative_base

DATABASE_URL = "sqlite+aiosqlite:///./prod.db"

engine = create_async_engine(DATABASE_URL, echo=False, future=True)

async_session_maker = sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)

Base = declarative_base()

async def get_async_session():
    async with async_session_maker() as session:
        yield session
```

### Test File with Dependency Override

``` python
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

from app.main import app
from app.database import Base, get_async_session
from app.models import User

# Create TEST DB (in memory)
TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"

test_engine = create_async_engine(TEST_DATABASE_URL, echo=False, future=True)

TestSessionLocal = sessionmaker(
    test_engine, expire_on_commit=False, class_=AsyncSession
)


async def override_get_async_session():
    async with TestSessionLocal() as session:
        yield session


app.dependency_overrides[get_async_session] = override_get_async_session


@pytest.fixture(scope="module", autouse=True)
async def prepare_database():
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest.mark.asyncio
async def test_create_user():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post("/users", params={"username": "alice", "job": "Engineer"})

    assert response.status_code == 200
    assert response.json()["message"] == "User created"


@pytest.mark.asyncio
async def test_get_user():
    async with TestSessionLocal() as session:
        session.add(User(username="bob", job="Designer"))
        await session.commit()

    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/users/bob")

    assert response.status_code == 200
    assert response.json() == {"username": "bob", "job": "Designer"}
```

How this works:

- Uses in-memory async SQLite — fast for tests, isolated from production

- FastAPI’s `Depends(get_async_session)` is replaced with the test session

- No real database I/O; everything runs in RAM

- Tests are fully asynchronous with `pytest.mark.asyncio`

## JWT Authentication Tests

### JWT Utility (`auth.py`)

``` python
from datetime import datetime, timedelta
from typing import Optional
import jwt

SECRET_KEY = "TEST_SECRET_KEY"
ALGORITHM = "HS256"


def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
```

### FastAPI App with JWT Auth (`main.py`)

``` python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
import jwt

from app.auth import SECRET_KEY, ALGORITHM

app = FastAPI()

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


def verify_token(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")


@app.get("/secure")
async def secure_endpoint(payload: dict = Depends(verify_token)):
    return {"message": "Access granted", "payload": payload}
```

### JWT Test Cases

``` python
import pytest
from datetime import timedelta
from httpx import AsyncClient
from app.main import app
from app.auth import create_access_token


@pytest.mark.asyncio
async def test_valid_jwt():
    token = create_access_token({"sub": "alice"}, expires_delta=timedelta(minutes=5))

    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/secure", headers={"Authorization": f"Bearer {token}"})

    assert response.status_code == 200
    assert response.json()["message"] == "Access granted"
    assert response.json()["payload"]["sub"] == "alice"


@pytest.mark.asyncio
async def test_expired_jwt():
    token = create_access_token({"sub": "user"}, expires_delta=timedelta(minutes=-1))

    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/secure", headers={"Authorization": f"Bearer {token}"})

    assert response.status_code == 401
    assert response.json() == {"detail": "Token expired"}


@pytest.mark.asyncio
async def test_missing_token():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/secure")

    assert response.status_code == 401
    assert response.json()["detail"] == "Not authenticated"


@pytest.mark.asyncio
async def test_invalid_token():
    invalid_token = "abc.def.ghi"

    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get(
            "/secure",
            headers={"Authorization": f"Bearer {invalid_token}"}
        )

    assert response.status_code == 401
    assert response.json() == {"detail": "Invalid token"}


@pytest.mark.asyncio
async def test_wrong_secret_jwt():
    fake_token = jwt.encode({"sub": "hacker"}, "WRONG_KEY", algorithm="HS256")

    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/secure", headers={"Authorization": f"Bearer {fake_token}"})

    assert response.status_code == 401
    assert response.json() == {"detail": "Invalid token"}
```

### Test Coverage Summary

| Test                    | Validates                         |
|-------------------------|-----------------------------------|
| `test_valid_jwt`        | Proper access to protected routes |
| `test_expired_jwt`      | Token expiration handling         |
| `test_missing_token`    | OAuth2 missing header logic       |
| `test_invalid_token`    | Token structure errors            |
| `test_wrong_secret_jwt` | Signature mismatch handling       |

## Recommended Test Folder Structure

```
tests/
│
├── __init__.py
│
├── conftest.py                 # Global pytest fixtures (DB, clients, settings)
│
├── factories/                  # Data builders / model factories
│   ├── __init__.py
│   └── user_factory.py
│
├── utils/                      # Shared testing utilities
│   ├── __init__.py
│   └── jwt_helpers.py
│
├── unit/                       # Pure unit tests (no DB, no FastAPI)
│   ├── __init__.py
│   └── test_helpers.py
│
├── integration/                # API + DB tests, Async tests
│   ├── __init__.py
│   ├── test_users.py
│   └── test_auth.py
│
└── e2e/                        # End-to-end tests (simulate full behavior)
    ├── __init__.py
    └── test_full_flow.py
```

| Folder         | Purpose                      |
|----------------|------------------------------|
| `unit/`        | Fast, isolated logic tests   |
| `integration/` | DB + API + Async tests       |
| `e2e/`         | Full workflow scenarios      |
| `factories/`   | Model and payload generators |
| `utils/`       | Reusable test helpers        |
| `conftest.py`  | Global fixtures              |

------------------------------------------------------------------------

## Essential pytest Plugins (2026 Update)

### pytest-asyncio — Async Test Support

Required for testing async FastAPI endpoints with `AsyncClient`:

``` bash
pip install pytest-asyncio
```

``` python
import pytest
from httpx import AsyncClient, ASGITransport
from main import app

@pytest.mark.asyncio
async def test_create_user():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.post("/users", json={"name": "Alice", "email": "[email protected]"})
        assert response.status_code == 201
```

Configure in `pyproject.toml`:

``` toml
[tool.pytest.ini_options]
asyncio_mode = "auto"  # All async test functions are automatically treated as asyncio
```

### pytest-cov — Coverage Reporting

``` bash
# Run tests with coverage
pytest --cov=src --cov-report=term-missing --cov-fail-under=80

# Generate HTML coverage report
pytest --cov=src --cov-report=html
```

Integrate in CI (GitLab CI example):

``` yaml
test:
  script:
    - pytest --cov=src --cov-report=term-missing --cov-fail-under=80
  coverage: '/TOTAL.*\s+(\d+%)/'
```

### pytest-mock — Enhanced Mocking

Provides a `mocker` fixture that is cleaner than `unittest.mock.patch`:

``` python
def test_send_email(mocker):
    # Mock the email service
    mock_send = mocker.patch("services.email.send_email")
    mock_send.return_value = True

    result = notify_user(user_id=1)

    assert result is True
    mock_send.assert_called_once()
```

### Plugin Summary

| Plugin | Purpose | Install |
|----|----|----|
| `pytest-asyncio` | Async test support | `pip`` ``install`` ``pytest-asyncio` |
| `pytest-cov` | Coverage reporting | `pip`` ``install`` ``pytest-cov` |
| `pytest-mock` | Enhanced mocking | `pip`` ``install`` ``pytest-mock` |
| `pytest-xdist` | Parallel test execution | `pip`` ``install`` ``pytest-xdist` |
| `httpx` | Async HTTP client for testing | `pip`` ``install`` ``httpx` |
