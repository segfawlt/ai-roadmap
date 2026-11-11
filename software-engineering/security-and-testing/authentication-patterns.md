# Authentication Implementation Overview

``` mermaid
        sequenceDiagram
    participant Client as Client
    participant FastAPI as FastAPI App
    participant AuthServer as AuthServer
    participant DB as User Store
    Client->>FastAPI: POST /token (username + password form)
    FastAPI->>AuthServer: issue_token(username, password)
    AuthServer->>DB: authen_user(username, password)
    DB-->>AuthServer: user dict or None
    AuthServer-->>FastAPI: JWT access token
    FastAPI-->>Client: {access_token, token_type: bearer}
    Client->>FastAPI: POST /protected (Authorization: Bearer token)
    FastAPI->>AuthServer: verify_token(token)
    AuthServer-->>FastAPI: decoded claims
    FastAPI-->>Client: protected resource
```

Authentication in FastAPI typically involves:

- Verifying user credentials (username/password)

- Issuing JWT tokens

- Protecting routes by validating tokens on each request

- Integrating with FastAPI’s dependency injection system

## Build `AuthServer`

### Install required packages

``` bash
pip install python-jose passlib[bcrypt]
```

### Implement base `AuthServer`

``` python
from datetime import datetime, timedelta
from jose import JWTError, jwt


class AuthServer:
    def __init__(self):
        self._access_token_expire_mins = 30
        self._secret_key = "supersecretkey123"
        self._algorithm = "HS256"

    def authen_user(self, *args, **kwargs) -> dict:
        raise NotImplementedError()

    def issue_token(self, *args, **kwargs):
        user_dict = self.authen_user(*args, **kwargs)
        if user_dict is None:
            raise Exception("User not authenticated")

        to_encode = user_dict.copy()
        expire = datetime.now() + timedelta(minutes=self._access_token_expire_mins)
        to_encode.update({"exp": expire})
        return jwt.encode(to_encode, self._secret_key, algorithm=self._algorithm)

    def verify_token(self, token: str) -> dict:
        return jwt.decode(token, self._secret_key, algorithms=[self._algorithm])


class FakeAuthServer(AuthServer):
    def authen_user(self, *args, **kwargs):
        return kwargs


# Sample test
auth_server = FakeAuthServer()
token = auth_server.issue_token(user_id="HoHai", password="123")
print(f"Token: {token}")
print(f"Decode token: {auth_server.verify_token(token)}")
```

### Implement `AuthUserPassServer`

``` python
import inspect
import asyncio
import nest_asyncio

class AuthUserPassServer(AuthServer):
    def __init__(self, fn_check_userpass):
        super().__init__()
        assert fn_check_userpass is not None, "Must set `fn_check_userpass`"
        self.fn = fn_check_userpass

    def authen_user(self, username, password) -> dict:
        # Support async functions
        if inspect.iscoroutinefunction(self.fn):
            nest_asyncio.apply()
            loop = asyncio.get_event_loop()
            task = loop.create_task(self.fn(username, password))
            loop.run_until_complete(task)
            return task.result()
        else:
            return self.fn(username, password)


# Sample test
auth_server = AuthUserPassServer(fn_check_userpass=lambda u, p: {"user": u, "password": p})
token = auth_server.issue_token(username="hohai", password="13")
print(f"Token: {token}")
print(f"Decode token: {auth_server.verify_token(token)}")
```

## Apply in FastAPI (Minimal)

``` python
import uvicorn
from fastapi import FastAPI, Depends
from fastapi.security import OAuth2PasswordRequestForm
import nest_asyncio
import threading

nest_asyncio.apply()

app = FastAPI()

auth_server = AuthUserPassServer(fn_check_userpass=lambda u, p: {"user": u, "password": p})


@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    token = auth_server.issue_token(form_data.username, form_data.password)
    return {"access_token": token, "token_type": "bearer"}

@app.post("/me")
async def verify(token: str):
    return auth_server.verify_token(token)


def run_app():
    uvicorn.run(app, host="0.0.0.0", port=8000)

thread = threading.Thread(target=run_app, daemon=True, name="Thread run FASTAPI")
thread.start()
```

``` mermaid
        graph TD
    Req[POST /token<br>Content-Type: application/x-www-form-urlencoded<br>grant_type=password<br>username=johndoe<br>password=secret] --> DI[FastAPI Depends<br>OAuth2PasswordRequestForm]
    DI --> Parse[Parse form fields<br>username password scope<br>grant_type client_id client_secret]
    Parse --> Handler[login form_data<br>form_data.username<br>form_data.password]
    Handler --> Token[return access_token]
    style DI fill:#9cf,stroke:#333
    style Parse fill:#9f9,stroke:#333
```

## What is `OAuth2PasswordRequestForm` in FastAPI

### In short

It’s a **dependency class** that parses and validates a login form sent with these fields:

- `username`

- `password`

- (optional) `scope`

- (optional) `grant_type`

- (optional) `client_id`

- (optional) `client_secret`

### Where it comes from

``` python
from fastapi.security import OAuth2PasswordRequestForm
```

Typical usage:

``` python
from fastapi import FastAPI, Depends
from fastapi.security import OAuth2PasswordRequestForm

app = FastAPI()

@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user_dict = {"username": form_data.username}
    # verify username & password here...
    return {"access_token": user_dict["username"], "token_type": "bearer"}
```

### Expected request format

The client must send the data as **`application/x-www-form-urlencoded`**, not JSON:

```
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=password&username=johndoe&password=secret
```

### Why it’s useful

- Automatically reads and validates form data.

- Matches the OAuth2 “Resource Owner Password Credentials” grant specification.

- Works seamlessly with FastAPI’s OAuth2 and security utilities.

## Password Hashing

Use `passlib` or `bcrypt` for secure hashing. Never store plain passwords.

``` python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str):
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str):
    return pwd_context.verify(plain_password, hashed_password)
```

## Role-Based Access Control (RBAC)

``` python
async def require_role(role: str):
    async def role_checker(current_user = Depends(get_current_user)):
        if current_user.role != role:
            raise HTTPException(status_code=403, detail="Not enough permissions")
    return role_checker

@app.get("/admin")
async def admin_dashboard(role_check = Depends(require_role("admin"))):
    return {"msg": "Welcome Admin"}
```

## CORS Configuration

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
