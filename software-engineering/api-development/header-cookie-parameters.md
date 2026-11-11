# Header Parameters in FastAPI

In **FastAPI**, a **Header parameter** is used to extract and validate values from the **HTTP request headers**. These are often used for things like:

- Authentication tokens (e.g., `Authorization`)

- Custom headers (e.g., `X-Request-ID`)

- Content negotiation (e.g., `Accept`, `User-Agent`)

## Basic Example

``` python
from fastapi import FastAPI, Header

app = FastAPI()

@app.get("/items/")
def read_items(user_agent: str = Header(None)):
    return {"User-Agent": user_agent}
```

- This reads the `User-Agent` header from the request.

- `Header(None)` makes it optional.

- If not provided, `user_agent` will be `None`.

## Required Header Example

``` python
@app.get("/secure-data/")
def get_secure_data(api_key: str = Header(...)):
    return {"API-Key": api_key}
```

- `...` makes the `api_key` header **required**

- If missing, FastAPI returns a **422 error**

## Header Name Conventions

- HTTP headers are **case-insensitive** and use **hyphens** (e.g., `X-Token`)

- FastAPI automatically converts **underscore `_` to hyphen `-`**

``` python
@app.get("/check/")
def check_header(x_token: str = Header(...)):
    return {"X-Token": x_token}
```

This will correctly read the `X-Token` header from the request.

## Example with Multiple Headers

``` python
@app.get("/info/")
def get_info(
    x_token: str = Header(...),
    x_request_id: str = Header(None)
):
    return {"token": x_token, "request_id": x_request_id}
```

## Summary: Parameter Types

| Parameter Type | Source              | Use Case                |
|----------------|---------------------|-------------------------|
| `Path`         | URL path            | Resource identification |
| `Query`        | URL query string    | Filtering, pagination   |
| `Body`         | Request body (JSON) | Structured data         |
| `Header`       | HTTP headers        | Metadata, auth, tracing |

## Cookie Parameters in FastAPI

In FastAPI, a **cookie parameter** is used to extract and validate values from the **HTTP cookies** sent by the client in the request. You use the `Cookie` class from `fastapi` to access these values.

## What Are Cookies?

Cookies are small pieces of data stored on the client side and sent with every HTTP request to the server. They’re often used for:

- Session management

- User preferences

- Tracking

## Basic Example: Reading a Cookie

``` python
from fastapi import FastAPI, Cookie

app = FastAPI()

@app.get("/read-cookie/")
def read_cookie(session_id: str = Cookie(None)):
    return {"session_id": session_id}
```

- `Cookie(None)` makes the `session_id` cookie optional.

- If the cookie is not present, `session_id` will be `None`.

## Required Cookie Example

``` python
@app.get("/secure/")
def secure_area(auth_token: str = Cookie(...)):
    return {"auth_token": auth_token}
```

- `...` means the cookie is **required**

- If the client doesn’t send the `auth_token` cookie, FastAPI returns a **422 error**

## Cookie Name Conversion

FastAPI converts **underscores to hyphens** in cookie names:

``` python
@app.get("/check/")
def check_cookie(user_id_cookie: str = Cookie(...)):
    return {"user_id_cookie": user_id_cookie}
```

This will read a cookie named `user-id-cookie` from the request.

## Session Auth with Cookies

The typical cookie-based session authentication flow:

1.  User logs in with credentials.

2.  Server verifies credentials and sets a session cookie (e.g., `session_id`).

3.  On subsequent requests, the client automatically sends the cookie.

4.  Server reads the cookie and validates the session.

``` python
from fastapi import FastAPI, Response, Cookie, HTTPException, status

app = FastAPI()

fake_session_store = {}

@app.post("/login")
def login(response: Response, username: str, password: str):
    if username == "admin" and password == "secret":
        session_id = "abc123"
        fake_session_store[session_id] = username
        response.set_cookie(key="session_id", value=session_id, httponly=True)
        return {"message": "Logged in"}
    raise HTTPException(status_code=401, detail="Invalid credentials")

@app.get("/profile")
def get_profile(session_id: str = Cookie(None)):
    if session_id in fake_session_store:
        return {"username": fake_session_store[session_id]}
    raise HTTPException(status_code=401, detail="Invalid or missing session")

@app.post("/logout")
def logout(response: Response, session_id: str = Cookie(None)):
    if session_id in fake_session_store:
        del fake_session_store[session_id]
    response.delete_cookie("session_id")
    return {"message": "Logged out"}
```

Key points:

- `response.set_cookie(...)`: sets the cookie on login

- `Cookie(...)`: reads the cookie from incoming requests

- `response.delete_cookie(...)`: removes the cookie on logout

- `httponly=True`: prevents JavaScript access to the cookie (for security)

## JWT in Cookies

A more secure approach stores JWT tokens in HTTP-only cookies:

1.  User logs in → server generates a **JWT token**

2.  Server sets the JWT in an **HTTP-only cookie**

3.  On future requests, the client sends the cookie automatically

4.  Server reads the cookie, verifies the JWT, and authenticates the user

``` python
from fastapi import FastAPI, Depends, HTTPException, status, Response, Cookie
from jose import JWTError, jwt
from datetime import datetime, timedelta
from typing import Optional

app = FastAPI()

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

fake_user = {"username": "admin", "password": "secret"}

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

@app.post("/login")
def login(response: Response, username: str, password: str):
    if username != fake_user["username"] or password != fake_user["password"]:
        raise HTTPException(status_code=401, detail="Invalid credentials")

    access_token = create_access_token(data={"sub": username})
    response.set_cookie(
        key="access_token",
        value=access_token,
        httponly=True,
        secure=False,  # Set to True in production with HTTPS
        samesite="lax"
    )
    return {"message": "Logged in"}

def get_current_user(access_token: Optional[str] = Cookie(None)):
    if not access_token:
        raise HTTPException(status_code=401, detail="Missing token")
    try:
        payload = jwt.decode(access_token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        return {"username": username}
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/profile")
def read_profile(user: dict = Depends(get_current_user)):
    return {"user": user}

@app.post("/logout")
def logout(response: Response):
    response.delete_cookie("access_token")
    return {"message": "Logged out"}
```

Key features:

- **JWT** is stored in a **secure, HTTP-only cookie**

- **No localStorage** or JavaScript access to the token

- **Automatic cookie handling** by the browser

- **Stateless**: no server-side session storage needed
