# Data Modeling (Pydantic)

This page explains how to use Pydantic’s `BaseModel` in FastAPI to define request bodies, apply validation rules, compose nested models, and control response output with `response_model`.

Topic cover:

- Request Body

- Pydantic BaseModel

- Advanced Validation

- Nested Models (Body)

- response_model to filter output

## 1. Introduction & Setup (15 min)

### Objectives

- Set up a FastAPI project environment.

- Understand what Pydantic and FastAPI do for data validation.

### Content

**overview:**

- **FastAPI** is a high-performance web framework for building APIs with Python 3.7+.

- It automatically handles **data parsing, validation, and documentation** using **Pydantic** and **OpenAPI**.

### Setup Instructions

``` bash
# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate  # (Windows: venv\Scripts\activate)

# Install dependencies
pip install fastapi uvicorn
```

### Starter App

``` python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello, FastAPI!"}
```

**Run the server:**

``` bash
uvicorn main:app --reload
```

### Exercise (5 min)

- Run the app and open: `http://127.0.0.1:8000/docs`

------------------------------------------------------------------------

## 2. Request Body (30 min)

### Objectives

- Learn how to handle incoming JSON request bodies.

- Understand FastAPI’s automatic request parsing.

### Content

#### Example: Handling a Simple Request Body

``` python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.post("/items/")
async def create_item(item: Item):
    return {"received": item}
```

**What happens:**

- FastAPI reads and parses the JSON body.

- Converts it into a Pydantic model (`Item`).

- Returns structured JSON automatically.

### Exercise (10 min)

**Task:** Create an endpoint `/users/` that accepts a user object with `username`, `email`, and optional `age`.

------------------------------------------------------------------------

## 3. Pydantic `BaseModel` (40 min)

### Objectives

- Understand how `BaseModel` works.

- Learn type hints, default values, and field customization.

### Key Concepts

- Type validation: `str`, `int`, `float`, `bool`, etc.

- Optional fields using `Optional` or `|`` ``None`.

- Field constraints via `Field`.

### Example: Using `Field` for Metadata

``` python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(..., min_length=3, max_length=50)
    price: float = Field(..., gt=0)
    in_stock: bool = Field(default=True)
```

### Example Endpoint

``` python
@app.post("/products/")
async def create_product(product: Product):
    return product
```

### Exercise (15 min)

- Create a model `Book` with fields:

  - `title` (min_length=2)

  - `author`

  - `pages` (must be \> 0)

  - `published` (bool, default `True`)

- POST `/books/` → return the same data.

------------------------------------------------------------------------

## 4. Advanced Validation (50 min)

### Objectives

- Use validators to apply custom validation logic.

- Learn root validators and complex data checks.

### Example: Field Validators

``` python
from pydantic import validator

class User(BaseModel):
    username: str
    password: str

    @validator("password")
    def password_strength(cls, v):
        if len(v) < 8:
            raise ValueError("Password must be at least 8 characters long")
        return v
```

### Example: Root Validators

``` python
from pydantic import root_validator

class Order(BaseModel):
    price: float
    quantity: int
    total: float

    @root_validator
    def check_total(cls, values):
        price, quantity, total = values.get('price'), values.get('quantity'), values.get('total')
        if total != price * quantity:
            raise ValueError("Total must equal price × quantity")
        return values
```

### Exercise (20 min)

- Build a model `Registration` with:

  - `email` (must contain “@”)

  - `password` (min 8 chars)

  - `confirm_password`

- Use a root validator to ensure passwords match.

------------------------------------------------------------------------

## 5. Nested Models (Body) (40 min)

### Objectives

- Learn how to use nested Pydantic models for complex request bodies.

- Understand how FastAPI automatically parses nested data.

### Example: Nested Models

``` python
class Address(BaseModel):
    city: str
    state: str
    zip_code: str

class UserProfile(BaseModel):
    name: str
    email: str
    address: Address

@app.post("/profile/")
async def create_profile(profile: UserProfile):
    return profile
```

### Exercise (20 min)

- Create a nested model for an **Order**:

  - `Customer` (name, email)

  - `Item` (name, price, quantity)

  - `Order` (customer: `Customer`, items: List\[`Item`\])

- POST `/orders/` → Return total order amount.

------------------------------------------------------------------------

## 6. Using `response_model` to Filter Output (40 min)

### Objectives

- Use `response_model` to structure or filter API responses.

- Hide sensitive fields such as passwords.

### Example

``` python
class UserIn(BaseModel):
    username: str
    password: str
    email: str

class UserOut(BaseModel):
    username: str
    email: str

@app.post("/users/", response_model=UserOut)
async def create_user(user: UserIn):
    # Simulate saving user to DB
    return user
```

**What happens:**

- The request accepts all fields from `UserIn`.

- The response **only returns** fields from `UserOut`.

### Exercise (15 min)

- Create:

  - `UserIn`: includes `password`

  - `UserOut`: excludes `password`

- POST `/users/register` → returns safe public data.

------------------------------------------------------------------------

## 7. Recap Practice (30 min)

### Summary

- FastAPI automatically validates request bodies.

- Pydantic models enforce structure and constraints.

- Validators allow for custom logic.

- Nested models handle complex payloads.

- `response_model` protects and structures output.

### Final Challenge (Optional)

Build a small API for a **Library System**:

- `Book`, `Author`, and `Publisher` models.

- POST `/books/` with nested models.

- Use `response_model` to hide internal IDs.

------------------------------------------------------------------------

## Pydantic v2.12+

### Conditional Field Exclusion (`exclude_if`)

Control when computed fields are included in the response:

``` python
from pydantic import BaseModel, computed_field


class UserProfile(BaseModel):
    name: str
    email: str
    is_admin: bool = False

    @computed_field(exclude_if=lambda self: not self.is_admin)
    @property
    def admin_panel_url(self) -> str:
        return f"/admin/users/{self.name}"


# Regular user: admin_panel_url is excluded from JSON output
# Admin user: admin_panel_url is included
```

### `ValidateAs` Type

A new way to validate values that need transformation:

``` python
from pydantic import BaseModel, ValidateAs


class Config(BaseModel):
    # Accept string or int, always store as int
    port: ValidateAs[int]  # "8080" → 8080, 8080 → 8080
    debug: ValidateAs[bool]  # "true" → True, 1 → True
```

### Performance Improvements

Pydantic v2.12+ includes significant internal optimizations:

- **2x faster JSON serialization** in FastAPI when `response_model` is set (uses Pydantic’s Rust-based serializer)

- Optimized `Literal` type validation (faster lookup)

- Faster model copying with `model_copy()`

Always set `response_model` on your FastAPI endpoints — it now enables the fast Rust serialization path in addition to filtering response fields.

------------------------------------------------------------------------

## Additional Resources

- [FastAPI Documentation](https://fastapi.tiangolo.com/)

- [Pydantic Documentation](https://docs.pydantic.dev/)

- [FastAPI Tutorial - Official](https://fastapi.tiangolo.com/tutorial/)

- [Pydantic Field Types and Validators](https://docs.pydantic.dev/latest/concepts/validators/)

- [Pydantic v2.12 Changelog](https://docs.pydantic.dev/latest/changelog/)
