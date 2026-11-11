# Advanced Assessment Quiz

This quiz covers all topics from the Advanced tier. Answer each question, then check your answers.

------------------------------------------------------------------------

## Section 1: JWT Fundamentals

**1. A JWT consists of three parts separated by dots. What are they?**

- A\) Username, Password, Signature

- B\) Header, Payload, Signature

- C\) Token, Claims, Hash

- D\) Key, Value, Expiration

Answer

**B** — A JWT has three Base64URL-encoded parts:

- **Header**: Algorithm and token type (`{"alg":`` ``"HS256",`` ``"typ":`` ``"JWT"}`)

- **Payload**: Claims — data about the user (`{"sub":`` ``"user_123",`` ``"exp":`` ``1234567890}`)

- **Signature**: Verification hash (`HMAC-SHA256(header`` ``+`` ``"."`` ``+`` ``payload,`` ``secret)`)

**2. What is the difference between HS256 and RS256 signing algorithms?**

Answer

| Algorithm | Type | Key | Use Case |
|----|----|----|----|
| **HS256** | Symmetric | Same secret for signing and verification | Single-service applications |
| **RS256** | Asymmetric | Private key signs, public key verifies | Microservices, third-party verification |

HS256 is simpler — one secret key shared between issuer and verifier. RS256 is more secure for distributed systems — only the auth server has the private key; all other services verify with the public key.

**3. A developer stores `{"user_id":`` ``42,`` ``"password":`` ``"secret123",`` ``"role":`` ``"admin"}` in a JWT payload. What is wrong?**

Answer

**Never store sensitive data (passwords) in JWT payloads.** JWTs are Base64URL-encoded, NOT encrypted. Anyone can decode the payload without the secret key — the signature only prevents **tampering**, not **reading**. Store only non-sensitive identifiers: `{"sub":`` ``"42",`` ``"role":`` ``"admin"}`.

------------------------------------------------------------------------

## Section 2: OAuth2 Framework

**4. What problem does OAuth2 solve that simple username/password authentication does not?**

Answer

OAuth2 enables **delegated authorization** — a user can grant a third-party application limited access to their resources (on another service) without sharing their password. Example: “Sign in with Google” lets your app access the user’s Google profile without knowing their Google password.

**5. Explain the OAuth2 Authorization Code flow in order.**

Answer

1.  **User** clicks “Login with Google” on your app

2.  **Your app** redirects user to Google’s authorization endpoint with `client_id`, `redirect_uri`, `scope`

3.  **Google** shows consent screen; user approves

4.  **Google** redirects back to your app with an **authorization code**

5.  **Your app’s backend** exchanges the code for an **access token** (server-to-server call with `client_secret`)

6.  **Your app** uses the access token to call Google APIs (e.g., get user profile)

Key: The authorization code is exchanged server-side so the `client_secret` is never exposed to the browser.

**6. What is the difference between OAuth2 and OpenID Connect (OIDC)?**

- A\) They are the same thing

- B\) OAuth2 handles authorization; OIDC adds authentication (identity) on top of OAuth2

- C\) OIDC replaces OAuth2 entirely

- D\) OAuth2 is for APIs; OIDC is for websites

Answer

**B** — OAuth2 is an authorization framework (grants access to resources). OIDC is a layer on top of OAuth2 that adds authentication (proves who the user is). OIDC introduces the **ID Token** (a JWT containing user identity claims like `email`, `name`, `sub`).

------------------------------------------------------------------------

## Section 3: Authentication Patterns

**7. Implement a FastAPI dependency that extracts and validates a JWT from the Authorization header.**

Answer

``` python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    result = await db.execute(select(User).where(User.id == int(user_id)))
    user = result.scalars().first()
    if user is None:
        raise credentials_exception
    return user
```

**8. Why should access tokens have a short expiration (e.g., 15 minutes) and refresh tokens have a longer expiration (e.g., 7 days)?**

Answer

**Access tokens** are sent with every API request and stored in potentially vulnerable locations (browser memory, mobile storage). A short expiration limits the damage if stolen — the attacker only has 15 minutes of access.

**Refresh tokens** are used only to get new access tokens and should be stored more securely (HTTPOnly cookies, server-side). They have longer expiration to avoid forcing users to re-login frequently.

If an access token is stolen: attacker has 15 minutes. If a refresh token is stolen: revoke it server-side and force re-login.

------------------------------------------------------------------------

## Section 4: Unit Testing

**9. What is the purpose of `TestClient` in FastAPI testing?**

Answer

`TestClient` (from `httpx` or `starlette.testclient`) sends HTTP requests to your FastAPI app **without starting a real server**. It simulates the full request/response cycle (routing, validation, dependency injection, middleware) in-memory, making tests fast and reliable.

``` python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_get_users():
    response = client.get("/users")
    assert response.status_code == 200
    assert isinstance(response.json(), list)
```

**10. How do you test a protected endpoint that requires JWT authentication?**

Answer

Two approaches:

**Approach 1: Override the dependency**

``` python
async def override_get_current_user():
    return User(id=1, email="[email protected]", name="Test User")

app.dependency_overrides[get_current_user] = override_get_current_user
```

**Approach 2: Send a real token**

``` python
def test_protected_endpoint():
    # Create a valid token for testing
    token = jwt.encode({"sub": "1", "exp": datetime.utcnow() + timedelta(hours=1)}, SECRET_KEY)
    response = client.get("/tickets", headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 200
```

Approach 1 is simpler for most tests. Approach 2 is needed when testing the auth flow itself.

**11. How do you mock an async database session in tests?**

Answer

Use a separate test database with dependency override:

``` python
# conftest.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

test_engine = create_async_engine("sqlite+aiosqlite:///./test.db")

async def override_get_db():
    async with AsyncSession(test_engine) as session:
        yield session

@pytest.fixture(autouse=True)
def setup_db():
    app.dependency_overrides[get_db] = override_get_db
    # Create tables before tests
    async def init():
        async with test_engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)
    asyncio.run(init())
    yield
    # Cleanup
    app.dependency_overrides.clear()
```

**12. Write a test that verifies creating a ticket returns 201 and the correct data.**

Answer

``` python
def test_create_ticket():
    # Arrange: Override auth dependency
    app.dependency_overrides[get_current_user] = lambda: User(id=1, name="Test")

    # Act: Create a ticket
    response = client.post("/api/v1/tickets", json={
        "content": "My laptop is not working",
        "description": "Screen is black after boot",
    })

    # Assert: Check status and response shape
    assert response.status_code == 201
    data = response.json()
    assert data["content"] == "My laptop is not working"
    assert data["status"] == "pending"
    assert "ticket_id" in data  # UUID was auto-generated
    assert data["user_id"] == 1  # Assigned from auth
```
