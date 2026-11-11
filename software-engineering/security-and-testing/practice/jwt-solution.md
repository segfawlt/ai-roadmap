# **Solutions for JWT Assignment**

## Task 2 — Create a Simple JWT\*\*

**File:** `jwt_create.py`

``` python
import jwt
import datetime

secret = "mysecret123"

# Payload with 1-minute expiration
payload = {
    "username": "alice",
    "iat": datetime.datetime.utcnow(),
    "exp": datetime.datetime.utcnow() + datetime.timedelta(minutes=1)
}

token = jwt.encode(payload, secret, algorithm="HS256")
print("Generated JWT:")
print(token)
```

------------------------------------------------------------------------

## Task 3 — Decode and Verify JWT\*\*

**File:** `jwt_verify.py`

``` python
import jwt
from jwt import ExpiredSignatureError, InvalidTokenError

secret = "mysecret123"

def verify_token(token):
    try:
        decoded = jwt.decode(token, secret, algorithms=["HS256"])
        print("Token is valid!")
        print("Decoded payload:", decoded)

    except ExpiredSignatureError:
        print("Token has expired.")

    except InvalidTokenError:
        print("Invalid token or signature.")

if __name__ == "__main__":
    t = input("Enter JWT: ")
    verify_token(t)
```
