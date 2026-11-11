# **FastAPI Advanced Topics: Authentication, Authorization, and Testing**

## **1. Introduction**

- overview of what we’ll cover:

  - OAuth2 with JWT

  - Password hashing

  - `get_current_user` dependency

  - Role-Based Access Control (RBAC)

  - Testing with Pytest and TestClient

  - CORS configuration

  - Async dependencies for efficiency

------------------------------------------------------------------------

## **2. Authentication with OAuth2 and JWT**

- **OAuth2 Password Flow**:

  - Explain why OAuth2 is used for token-based authentication.

  - Show how to use `OAuth2PasswordBearer` in FastAPI.

- **JWT (JSON Web Tokens)**:

  - Structure of JWT: Header, Payload, Signature.

  - Why JWT is stateless and efficient.

- **Implementation Steps**:

  - Install `python-jose` for JWT signing.

  - Create utility functions:

    - `create_access_token(data:`` ``dict,`` ``expires_delta:`` ``timedelta)`

    - `verify_token(token:`` ``str)`

- **Async Example**:

  ``` python
  from fastapi.security import OAuth2PasswordBearer
  from jose import JWTError, jwt

  oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

  async def get_current_user(token: str = Depends(oauth2_scheme)):
      try:
          payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
          username: str = payload.get("sub")
          if username is None:
              raise credentials_exception
          return username
      except JWTError:
          raise credentials_exception
  ```

------------------------------------------------------------------------

## **3. Password Hashing**

- Use `passlib` or `bcrypt` for secure hashing.

- Explain why hashing is critical (never store plain passwords).

- Example:

  ``` python
  from passlib.context import CryptContext

  pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

  def hash_password(password: str):
      return pwd_context.hash(password)

  def verify_password(plain_password: str, hashed_password: str):
      return pwd_context.verify(plain_password, hashed_password)
  ```

------------------------------------------------------------------------

## **4. Role-Based Access Control (RBAC)**

- Define roles: `admin`, `user`, etc.

- Implement a dependency that checks roles:

  ``` python
  async def require_role(role: str):
      async def role_checker(current_user = Depends(get_current_user)):
          if current_user.role != role:
              raise HTTPException(status_code=403, detail="Not enough permissions")
      return role_checker
  ```

- Usage:

  ``` python
  @app.get("/admin")
  async def admin_dashboard(role_check = Depends(require_role("admin"))):
      return {"msg": "Welcome Admin"}
  ```

------------------------------------------------------------------------

## **5. CORS Configuration**

- Why CORS matters for frontend-backend communication.

- Example:

  ``` python
  from fastapi.middleware.cors import CORSMiddleware

  app.add_middleware(
      CORSMiddleware,
      allow_origins=["*"],  # or specific domains
      allow_credentials=True,
      allow_methods=["*"],
      allow_headers=["*"],
  )
  ```

------------------------------------------------------------------------

## **6. Testing with Pytest and TestClient**

- Install `pytest` and `httpx` (or use FastAPI’s `TestClient`).

- Example test:

  ``` python
  from fastapi.testclient import TestClient
  from main import app

  client = TestClient(app)

  def test_read_main():
      response = client.get("/")
      assert response.status_code == 200
      assert response.json() == {"msg": "Hello World"}
  ```

- For async tests, use `httpx.AsyncClient`.

------------------------------------------------------------------------

## **7. Emphasizing Async Dependencies**

- Explain why `async`` ``def` improves scalability.

- Show how to make DB calls and external API calls async.

- Example:

  ``` python
  async def get_db():
      async with async_session() as session:
          yield session
  ```

------------------------------------------------------------------------

## **8. Putting It All Together**

- Build a small project:

  - `/register` → hash password and store user.

  - `/login` → issue JWT token.

  - `/me` → get current user.

  - `/admin` → RBAC check.

- Include tests and CORS setup.

------------------------------------------------------------------------

✅ Would you like me to **create a full sample FastAPI project** with all these features (OAuth2, JWT, hashing, RBAC, async DB, tests, and CORS) in a **single codebase**? Or should I **prepare a slide deck outline** for teaching these concepts?
