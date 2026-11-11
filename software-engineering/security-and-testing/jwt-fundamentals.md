# What is `JWT` ?

``` mermaid
        graph LR
    JWT[JWT Token<br>header.payload.signature] --> H[Header<br>Base64URL encoded<br>alg: HS256<br>typ: JWT]
    JWT --> P[Payload<br>Base64URL encoded<br>sub: user_id<br>exp: expiry<br>roles: user]
    JWT --> S[Signature<br>HMAC of header + payload<br>using secret key]
    style H fill:#f96,stroke:#333
    style P fill:#9cf,stroke:#333
    style S fill:#9f9,stroke:#333
```

- JWT stands for **JSON Web Token**. It’s an open standard (RFC 7519) used for securely transmitting information between parties as a JSON object.

- JWT is a token format often used **after authentication to maintain sessions**.

## **Key Points About JWT**

- **Structure**: A JWT consists of three parts:

  1.  **Header**: Specifies the token type (JWT) and the signing algorithm (e.g., HS256).

  2.  **Payload**: Contains the claims (data), such as user ID, roles, or expiration time.

  3.  **Signature**: Ensures the token hasn’t been tampered with. It’s created by hashing the header and payload with a secret key.

- **Format**: `header.payload.signature` (each part is Base64URL encoded).

- **Purpose**: Commonly used for **authentication and authorization** in web applications. For example:

  - After a user logs in, the server issues a JWT.

  - The client sends this token in the `Authorization` header for subsequent requests.

  - The server verifies the token instead of storing session data.

- **Advantages**:

  - **Stateless**: No need to store session on the server.

  - **Compact**: Easy to transmit via URL, header, or body.

  - **Secure**: If signed and optionally encrypted properly.

- **Common Use Cases**:

  - API authentication.

  - Single Sign-On (SSO).

  - Mobile and SPA (Single Page Application) authentication.

------------------------------------------------------------------------

Do you want me to **show an example of how to create and verify a JWT in Python using `PyJWT`**, or explain **how JWT differs from OAuth tokens**?

## Minimal sample using JWT (Using HS256)

- `HS256`: Share secret key (less secure)

``` mermaid
        sequenceDiagram
    participant App as Application
    participant JWT as PyJWT Library
    App->>JWT: jwt.encode(payload, SECRET_KEY, algorithm="HS256")
    JWT-->>App: token (header.payload.HMAC_signature)
    Note over App,JWT: Both sides share the same SECRET_KEY
    App->>JWT: jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    JWT-->>App: decoded claims dict
```

### Install `PyJWT` package

``` sh
pip install PyJWT
```

``` bash
!pip install PyJWT
```

### Implement issue JWT (encode)

``` python
import jwt
from datetime import datetime, timedelta, timezone

# Algorithm to encrype
ALGORITHM = "HS256"
# Use a strong secret from env/config
SECRET_KEY = "super-strong-secret-change-me"

def create_jwt(user_id: str) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "user_id": user_id,                 # subject (user id)
        "issue_date": int(now.timestamp()),    # issued at
        "expired": int((now + timedelta(minutes=15)).timestamp()),  # expiration
        "iss": "your-app",              # issuer
        "aud": "your-client",           # audience
        "roles": ["user"]               # custom claim
    }
    token = jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
    return token

token = create_jwt("haiht")
print("JWT (HS256):", token)
```

### Verify a token (decode)

``` python
import jwt



def verify_jwt(token: str) -> dict:
    try:
        decoded = jwt.decode(
            token,
            SECRET_KEY,
            algorithms=["HS256"],
            audience="your-client",
            issuer="your-app",
        )
        return decoded  # dict with claims
    except jwt.ExpiredSignatureError:
        raise ValueError("Token has expired.")
    except jwt.InvalidAudienceError:
        raise ValueError("Invalid audience.")
    except jwt.InvalidIssuerError:
        raise ValueError("Invalid issuer.")
    except jwt.InvalidTokenError as e:
        raise ValueError(f"Invalid token: {e}")

# Example usage
# claims = verify_jwt("haiht")

print(f"Token above: {token}")

claims = verify_jwt(token)

print("Verified claims:", claims)
```

## Minimal using (Public/Private key) Example

- Using `RS256`

``` mermaid
        sequenceDiagram
    participant Issuer as Issuer (Server)
    participant Client as Client
    participant Verifier as Verifier (Any party)
    Issuer->>Issuer: jwt.encode(payload, PRIVATE_KEY, algorithm="RS256")
    Issuer-->>Client: token (header.payload.RSA_signature)
    Client->>Verifier: send token
    Note over Verifier: Only needs PUBLIC_KEY to verify
    Verifier->>Verifier: jwt.decode(token, PUBLIC_KEY, algorithms=["RS256"])
    Verifier-->>Client: decoded claims dict
```

### Install extra encryption package

``` sh
pip install "pyjwt[crypto]"
```

``` bash
!pip install "pyjwt[crypto]"
```

### Create `public/private` key pair

``` sh
# Private key
openssl genrsa -out private.pem 2048
# Public key
openssl rsa -in private.pem -pubout -out public.pem
```

You can run directly in Jupyter Notebook by adding `!`

``` bash
# Private key
!openssl genrsa -out private.pem 2048
# Public key
!openssl rsa -in private.pem -pubout -out public.pem
```

### Encode with private key

``` python
import jwt
from datetime import datetime, timedelta, timezone

def read_file(path: str) -> str:
    with open(path, "r", encoding="utf-8") as f:
        return f.read()

PRIVATE_KEY = read_file("private.pem")

# print(PRIVATE_KEY)

def create_jwt_rs256(user_id: str) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "user_id": user_id,
        "": int(now.timestamp()),
        "exp": int((now + timedelta(minutes=15)).timestamp()),
        "iss": "your-app",
        "aud": "your-client",
        "scope": "read:profile",
    }
    token = jwt.encode(payload, PRIVATE_KEY, algorithm="RS256")
    return token


token = create_jwt_rs256("haiht")
print("JWT (RS256):", token)
```

```
JWT (RS256): eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiaGFpaHQiLCIiOjE3NjQ3NDQzMTQsImV4cCI6MTc2NDc0NTIxNCwiaXNzIjoieW91ci1hcHAiLCJhdWQiOiJ5b3VyLWNsaWVudCIsInNjb3BlIjoicmVhZDpwcm9maWxlIn0.bU6cqzAJqs0NiwGhqY5ByuhEwfLEiePAZMJbqz15BXnkZQHKePzClVUNWnMTw9XP8d2nWeekAU2AK1nA3i7KPoMPjBF650nvKIsnLOKwcS2GKWtcR8RNcYKXZJ4BCnrwRBv0H8Wq1ElCIjWknB7CqmYR36njYr5i6S1DmHLc58FUmgteO8SIC3HWX5YPwSxsxItkOOF4H-IIzFXsO0OnLD8X5CRo089Uu9E9rrBi_kIny7fyLKpkbsnkRA8Qr_QfLlAj4rZe_rGgVPCn-5BzM02uDhyWnstEe6ov6lwiDlCsHLlWizjxq2Jumt3BFVIUF76Jp-jYapeSwnT8r2LYnQ
```

### Decode with public key

``` python
import jwt

def read_file(path: str) -> str:
    with open(path, "r", encoding="utf-8") as f:
        return f.read()

PUBLIC_KEY = read_file("public.pem")

def verify_jwt_rs256(token: str) -> dict:
    try:
        decoded = jwt.decode(
            token,
            PUBLIC_KEY,
            algorithms=["RS256"],
            audience="your-client",
            issuer="your-app",
        )
        return decoded
    except jwt.ExpiredSignatureError:
        raise ValueError("Token has expired.")
    except jwt.InvalidAudienceError:
        raise ValueError("Invalid audience.")
    except jwt.InvalidIssuerError:
        raise ValueError("Invalid issuer.")
    except jwt.InvalidTokenError as e:
        raise ValueError(f"Invalid token: {e}")


# token = create_jwt_rs256("uhaiht")

print(f"Token above {token}")
claims = verify_jwt_rs256(token)
print("Verified claims:", claims)
```

```
Token above eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiaGFpaHQiLCIiOjE3NjQ3NDQzMTQsImV4cCI6MTc2NDc0NTIxNCwiaXNzIjoieW91ci1hcHAiLCJhdWQiOiJ5b3VyLWNsaWVudCIsInNjb3BlIjoicmVhZDpwcm9maWxlIn0.bU6cqzAJqs0NiwGhqY5ByuhEwfLEiePAZMJbqz15BXnkZQHKePzClVUNWnMTw9XP8d2nWeekAU2AK1nA3i7KPoMPjBF650nvKIsnLOKwcS2GKWtcR8RNcYKXZJ4BCnrwRBv0H8Wq1ElCIjWknB7CqmYR36njYr5i6S1DmHLc58FUmgteO8SIC3HWX5YPwSxsxItkOOF4H-IIzFXsO0OnLD8X5CRo089Uu9E9rrBi_kIny7fyLKpkbsnkRA8Qr_QfLlAj4rZe_rGgVPCn-5BzM02uDhyWnstEe6ov6lwiDlCsHLlWizjxq2Jumt3BFVIUF76Jp-jYapeSwnT8r2LYnQ
Verified claims: {'user_id': 'haiht', '': 1764744314, 'exp': 1764745214, 'iss': 'your-app', 'aud': 'your-client', 'scope': 'read:profile'}
```
