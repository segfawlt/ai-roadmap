# Secure Coding Practices

## Introduction

Security is not a feature you add later — it must be built into every line of code from the start. In 2026, with AI tools generating more code faster, security discipline is more critical than ever: AI-generated code can introduce vulnerabilities that look correct at a glance but contain subtle flaws.

This guide covers the essential secure coding practices every backend developer must know, based on the OWASP Secure Coding Practices and the OWASP Top 10.

**Key statistic:** Organizations adopting shift-left security (testing early in development) reduce production defects by 60-90% and cut the cost of fixing vulnerabilities by 40-60%.

------------------------------------------------------------------------

## The Security Mindset

### Trust Boundaries

Every system has trust boundaries — lines where data crosses from untrusted to trusted zones:

``` mermaid
        graph LR
    A[User Browser<br>UNTRUSTED] -->|HTTP Request| B[API Gateway]
    B -->|Validated Input| C[Application Logic<br>TRUSTED]
    C -->|Parameterized Query| D[Database<br>TRUSTED]
    E[External API<br>UNTRUSTED] -->|Response| C
```

**Rule:** Validate ALL data crossing a trust boundary. Never trust:

- User input (forms, URL parameters, headers, cookies)

- External API responses

- File uploads

- Data from message queues

- Environment variables from untrusted sources

------------------------------------------------------------------------

## Input Validation

The first and most important line of defense. Validate **type, length, format, and range** of every input.

### Pydantic Validation (FastAPI)

``` python
from pydantic import BaseModel, Field, EmailStr, field_validator
from datetime import datetime


class CreateUserRequest(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(ge=18, le=120)
    phone: str | None = Field(default=None, pattern=r"^\+?[0-9]{10,15}$")

    @field_validator("name")
    @classmethod
    def name_must_not_contain_html(cls, v: str) -> str:
        if "<" in v or ">" in v:
            raise ValueError("Name must not contain HTML tags")
        return v.strip()


class CreateBookingRequest(BaseModel):
    room_id: int
    start_time: datetime
    reason: str = Field(min_length=5, max_length=500)

    @field_validator("start_time")
    @classmethod
    def must_be_future(cls, v: datetime) -> datetime:
        if v <= datetime.utcnow():
            raise ValueError("Booking time must be in the future")
        return v
```

### Validation Checklist

| Check | Example | Why |
|----|----|----|
| **Type** | Is `age` an integer? | Prevents type confusion attacks |
| **Length** | Is `name` between 1-100 chars? | Prevents buffer overflow, DoS via huge payloads |
| **Range** | Is `quantity` between 1-10000? | Prevents business logic abuse |
| **Format** | Does `email` match email pattern? | Prevents injection via malformed input |
| **Allowlist** | Is `status` one of `["pending",`` ``"active"]`? | Prevents unexpected state transitions |

------------------------------------------------------------------------

## Injection Prevention

### SQL Injection

The most dangerous and most preventable vulnerability:

``` python
# BAD: String concatenation → SQL injection
query = f"SELECT * FROM users WHERE email = '{user_input}'"
# Attack: user_input = "'; DROP TABLE users; --"

# GOOD: Parameterized queries (SQLAlchemy)
from sqlalchemy import select

stmt = select(User).where(User.email == user_input)
result = session.execute(stmt).scalars().first()
```

**Rule:** NEVER concatenate user input into SQL strings. Always use parameterized queries or an ORM.

### Command Injection

``` python
import subprocess

# BAD: Shell injection
subprocess.run(f"ping {user_input}", shell=True)
# Attack: user_input = "8.8.8.8; rm -rf /"

# GOOD: Pass arguments as a list (no shell interpretation)
subprocess.run(["ping", "-c", "4", user_input], shell=False)
```

### Path Traversal

``` python
import os

# BAD: Directory traversal
file_path = f"/uploads/{user_filename}"
# Attack: user_filename = "../../etc/passwd"

# GOOD: Validate and resolve the path
upload_dir = os.path.realpath("/uploads")
requested = os.path.realpath(os.path.join(upload_dir, user_filename))
if not requested.startswith(upload_dir):
    raise ValueError("Invalid file path")
```

------------------------------------------------------------------------

## Authentication Security

### Password Storage

``` python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Hash a password (never store plaintext!)
hashed = pwd_context.hash("user_password")

# Verify a password
is_valid = pwd_context.verify("user_password", hashed)
```

**Rules:**

- Never store passwords in plaintext

- Use bcrypt, scrypt, or Argon2 (not MD5, not SHA-256)

- Never log passwords, even during debugging

- Implement account lockout after N failed attempts

### JWT Security

``` python
from jose import jwt

# Use strong secrets (at least 256 bits)
SECRET_KEY = os.environ["JWT_SECRET_KEY"]  # From environment, NEVER hardcoded

# Short expiration for access tokens
access_token = jwt.encode(
    {"sub": str(user_id), "exp": datetime.utcnow() + timedelta(minutes=15)},
    SECRET_KEY,
    algorithm="HS256",
)

# NEVER store sensitive data in JWT payload (it's encoded, not encrypted)
# BAD: {"password": "...", "ssn": "..."}
# GOOD: {"sub": "user_123", "role": "admin"}
```

------------------------------------------------------------------------

## Secrets Management

### The Rules

1.  **Never hardcode secrets** in source code

2.  **Never commit secrets** to Git (use `.gitignore` + pre-commit hooks)

3.  **Use environment variables** or a secret manager

4.  **Rotate secrets** regularly and immediately if exposed

5.  **Validate secrets exist** at application startup

### Environment Variables Pattern

``` python
# config.py
import os


class Settings:
    DATABASE_URL: str = os.environ["DATABASE_URL"]
    JWT_SECRET_KEY: str = os.environ["JWT_SECRET_KEY"]
    REDIS_URL: str = os.environ.get("REDIS_URL", "redis://localhost:6379")

    def __init__(self):
        # Fail fast if required secrets are missing
        required = ["DATABASE_URL", "JWT_SECRET_KEY"]
        missing = [key for key in required if key not in os.environ]
        if missing:
            raise RuntimeError(f"Missing required environment variables: {missing}")
```

### .env File (Development Only)

``` bash
# .env — NEVER commit this file
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/myapp
JWT_SECRET_KEY=your-super-secret-key-at-least-32-chars
REDIS_URL=redis://localhost:6379
```

```
# .gitignore — ALWAYS include
.env
.env.*
*.pem
*.key
secrets/
```

### Pre-Commit Secrets Detection

``` yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ["--baseline", ".secrets.baseline"]
```

------------------------------------------------------------------------

## Dependency Security

Third-party packages are a major attack vector. In 2026, supply chain attacks are increasingly common.

### Scanning Dependencies

``` bash
# Python: pip-audit
pip install pip-audit
pip-audit

# Python: Safety (alternative)
pip install safety
safety check

# Node.js: npm audit
npm audit
npm audit fix

# Universal: Trivy
trivy fs --scanners vuln .
```

### Dependency Best Practices

| Practice | Why |
|----|----|
| Pin exact versions in lockfiles | Prevent surprise updates from introducing vulnerabilities |
| Run `pip-audit` / `npm`` ``audit` in CI | Catch known CVEs before deployment |
| Update dependencies regularly | Don’t let vulnerabilities accumulate |
| Review new dependencies before adding | Check maintenance status, download count, known issues |
| Use minimal dependencies | Fewer deps = smaller attack surface |

------------------------------------------------------------------------

## Error Handling

### Safe Error Responses

``` python
from fastapi import HTTPException

# BAD: Leaks internal details
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    try:
        user = db.query(User).filter(User.id == user_id).first()
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
        # Exposes: "sqlalchemy.exc.OperationalError: connection refused to db:5432"

# GOOD: User-friendly message, log details server-side
import logging
logger = logging.getLogger(__name__)

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    try:
        user = db.query(User).filter(User.id == user_id).first()
    except Exception:
        logger.exception("Database error fetching user %s", user_id)
        raise HTTPException(status_code=500, detail="Internal server error")
```

**Rules:**

- Never expose stack traces, database errors, or internal paths to users

- Log detailed errors server-side for debugging

- Use generic error messages for 5xx responses

- Use specific, helpful messages for 4xx responses (validation errors)

------------------------------------------------------------------------

## Security Checklist

Run this checklist before every deployment:

### Input & Output

All user input validated (type, length, format, range)

SQL queries use parameterized statements or ORM

No string concatenation in shell commands

File paths validated against directory traversal

API responses do not expose sensitive fields

### Authentication & Authorization

Passwords hashed with bcrypt/scrypt/Argon2

JWT tokens have short expiration (15 min access, 7 day refresh)

Every endpoint checks authentication AND authorization

No sensitive data in JWT payload

### Secrets & Configuration

No hardcoded secrets in source code

`.env` files in `.gitignore`

Pre-commit hook detects secrets

Required secrets validated at startup

### Dependencies

All dependencies pinned in lockfile

`pip-audit` / `npm`` ``audit` passes in CI

No known critical CVEs in dependencies

### Infrastructure

HTTPS enforced (no HTTP in production)

CORS configured with specific origins (not `*`)

Rate limiting on authentication endpoints

Error messages do not leak internal details

------------------------------------------------------------------------

## References

1.  [OWASP Secure Coding Practices](https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/)

2.  [OWASP API Security Top 10 (2025)](https://owasp.org/API-Security/)

3.  [OWASP Top 10 (2025)](https://owasp.org/www-project-top-ten/)

4.  [FastAPI Security Documentation](https://fastapi.tiangolo.com/tutorial/security/)

5.  [Bandit — Python Security Linter](https://bandit.readthedocs.io/)

## Practice

### Exercise 1: Find the Vulnerabilities

Review this code and identify all security issues:

``` python
@app.post("/login")
async def login(username: str, password: str, db: Session = Depends(get_db)):
    query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
    user = db.execute(query).first()
    if user:
        token = jwt.encode(
            {"user_id": user.id, "password": password, "exp": datetime.utcnow() + timedelta(days=365)},
            "secret123",
            algorithm="HS256",
        )
        return {"token": token, "db_version": db.execute("SELECT version()").scalar()}
    raise HTTPException(status_code=401, detail=f"No user found with username {username}")
```

**Tasks:**

1.  List every security vulnerability (aim for 7+)

2.  Rewrite the endpoint following all secure coding practices from this guide

3.  Add input validation using Pydantic

### Exercise 2: Secrets Audit

Run a secrets audit on your project:

1.  Install `detect-secrets` and create a baseline

2.  Intentionally add a fake API key to a file

3.  Verify the pre-commit hook catches it

4.  Check your `.gitignore` covers all sensitive file patterns

### Exercise 3: Dependency Security

1.  Run `pip-audit` on your project

2.  If vulnerabilities are found, upgrade the affected packages

3.  Add `pip-audit` to your CI pipeline

4.  Document the process for handling a newly discovered CVE in a dependency

## Review Questions

1.  **What is SQL injection and how do you prevent it in Python?**

    - *Hint*: Never concatenate user input into queries. Use parameterized queries or ORM.

2.  **Why should passwords be hashed with bcrypt instead of SHA-256?**

    - *Hint*: Think about brute-force attack speed and the purpose of a “work factor.”

3.  **A developer stores the JWT secret key directly in `config.py` as `SECRET`` ``=`` ``"my-secret"`. What are the risks and how should it be fixed?**

    - *Hint*: What happens when this code is pushed to Git?

4.  **What is the difference between authentication and authorization? Give an API example.**

    - *Hint*: Authentication = “Who are you?” Authorization = “Are you allowed to do this?”

5.  **Why should error responses never include stack traces or database error messages?**

    - *Hint*: What information could an attacker learn from a detailed error message?

6.  **What is a supply chain attack? How do you protect against it?**

    - *Hint*: Think about third-party packages and what happens if one is compromised.
