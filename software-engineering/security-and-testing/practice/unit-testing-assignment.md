# **Assignment 1 — Build a Full Test Suite for a FastAPI App**

## **Goal:**

Create a complete **test suite** for a small FastAPI application. Students will practice:

- Unit tests

- Async API tests

- JWT authentication tests

- Mocked async DB testing with SQLAlchemy

- Applying a professional test folder structure

------------------------------------------------------------------------

## **Assignment Description**

You are given a FastAPI application (`from`` ``Unit`` ``06`) that contains:

1.  CRUD endpoints for users

2.  Async SQLAlchemy database connection

3.  JWT-protected endpoint

4.  Helper utilities

Your task is to create a **production-quality test suite** using:

- `pytest`

- `pytest-asyncio`

- `httpx.AsyncClient`

- `PyJWT`

- Mocked in-memory SQLite async DB

- Test folder structure best practices

------------------------------------------------------------------------

## **Requirements**

## **1. Create the test folder structure**

Your project test can be referred to as below (depending on the structure of Unit 06)

```
tests/
 ├── conftest.py
 ├── factories/
 │    └── user_factory.py
 ├── utils/
 │    └── jwt_helpers.py
 ├── unit/
 │    └── test_helpers.py
 ├── integration/
 │    ├── test_users.py
 │    └── test_auth.py
 └── e2e/
      └── test_full_flow.py
```

------------------------------------------------------------------------

## **2. Write tests for the following:**

### **A. UNIT TESTS (no FastAPI, no DB)**

Example: math helpers, pure functions, etc.

### **B. ASYNC INTEGRATION TESTS**

Using `AsyncClient`:

- Test async GET and POST endpoints

- Test error paths (404, 400, etc.)

### **C. MOCKED ASYNC SQLALCHEMY DB TESTS**

Using an **in-memory SQLite database**.

Example tasks:

- Mock DB session in `conftest.py`

- Test database CRUD operations via API

- Validate isolation between tests

### **D. JWT AUTH TESTS**

Test the `/secure` endpoint with:

- Valid token

- Expired token

- Missing authorization header

- Invalid/malformed token

- Wrong secret-key token

------------------------------------------------------------------------

## **3. End-to-end Test**

Simulate a full workflow:

1.  Create a user

2.  Authenticate (or generate a token)

3.  Access a secured resource

4.  Fetch user data from DB

------------------------------------------------------------------------

## **Expected Output**

A fully working test suite that can run using:

``` bash
pytest -v
```

------------------------------------------------------------------------

## **Grading Criteria**

| Category                   | Points |
|----------------------------|--------|
| Folder structure           | 10     |
| Unit tests                 | 10     |
| Async API tests            | 20     |
| Mocked DB tests            | 25     |
| JWT tests                  | 25     |
| Code quality & readability | 10     |

Total: **100 points**
