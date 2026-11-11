# **Assignment: Working With JWT in Python**

## **1. Objective**

By the end of this assignment, you should be able to:

- Understand what JWTs are and why they are used

- Create and verify JWTs in Python

- Implement JWT-based authentication in a simple API

- Handle expiration and errors securely

------------------------------------------------------------------------

## **2. Background Theory (Short Summary)**

A **JSON Web Token (JWT)** is a compact, URL-safe token used for authentication and authorization. A JWT has three parts:

```
header.payload.signature
```

- **Header** – declares the algorithm (e.g., HS256) and token type

- **Payload** – contains claims (user data, expiry time, etc.)

- **Signature** – verifies the token was not tampered with

Common Python libraries for JWT include:

- `pyjwt` (most common)

- `python-jose`

- `authlib`

## **3. Tasks**

### **Task 1: Install Dependencies**

Install PyJWT:

``` bash
pip install PyJWT
```

------------------------------------------------------------------------

### **Task 2: Create a Simple JWT**

Write a Python script that:

1.  Imports the PyJWT library

2.  Creates a payload with:

    - username

    - issued time (`iat`)

    - expiration time (`exp`)

3.  Signs the token using a secret key

4.  Prints the token

**Example structure:**

```
import jwt
import datetime

secret = "mysecret123"

payload = {
    "username": "alice",
    "iat": datetime.datetime.utcnow(),
    "exp": ???
}

token = jwt.encode(payload, secret, algorithm="HS256")
print(token)
```

**Your job:** calculate expiration to be 1 minute in the future.

------------------------------------------------------------------------

### **Task 3: Decode and Verify JWT**

Create another script or function that:

- Accepts a JWT token

- Verifies the signature

- Checks the expiration

- Prints the decoded payload

Example:

``` python
decoded = jwt.decode(token, secret, algorithms=["HS256"])
```

**Your job:** handle exceptions (`ExpiredSignatureError`, `InvalidTokenError`) and print custom messages.
