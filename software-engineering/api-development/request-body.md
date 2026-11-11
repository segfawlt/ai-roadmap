# `Body` in FastAPI

In **FastAPI**, `Body` is used to declare and validate **request body parameters** — the data sent by the client in the **body of the HTTP request**, typically in **JSON** format.

------------------------------------------------------------------------

## What Is a Request Body?

A **request body** is used when the client sends structured data to the server, such as:

- Creating or updating a resource

- Sending form data or JSON payloads

Example:

```
POST /users/
{
  "name": "Alice",
  "age": 30
}
```

------------------------------------------------------------------------

## Basic Example Using `Body`

``` python
from fastapi import FastAPI, Body

app = FastAPI()

@app.post("/users/")
def create_user(name: str = Body(...), age: int = Body(...)):
    return {"name": name, "age": age}
```

- `Body(...)` marks the parameter as **required**

- FastAPI will:

  - Parse the JSON body

  - Validate types

  - Return a 422 error if validation fails

------------------------------------------------------------------------

## Preferred Way: Use a Pydantic Model

``` python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int

@app.post("/users/")
def create_user(user: User):
    return user
```

- FastAPI automatically:

  - Parses the JSON body into a `User` object

  - Validates the fields

  - Generates OpenAPI docs

------------------------------------------------------------------------

## Advanced `Body` Options

You can customize the body input with:

``` python
from fastapi import Body

@app.post("/items/")
def create_item(
    name: str = Body(..., embed=True, description="The name of the item", example="Book")
):
    return {"name": name}
```

- `embed=True`: wraps the value in a key (e.g., `{"name":`` ``"Book"}` instead of just `"Book"`)

- `description`, `example`: used in API docs

------------------------------------------------------------------------

## Summary

| Parameter Type | Use | Example |
|----|----|----|
| `Path` | Required in URL path | `/users/{id}` |
| `Query` | Optional or required in URL query | `?search=abc` |
| `Body` | JSON or form data in request body | `{`` ``"name":`` ``"Alice"`` ``}` |

------------------------------------------------------------------------

In **FastAPI**, **Body Nested** refers to the use of **nested Pydantic models** to represent **complex JSON structures** in the request body. This is useful when your API expects structured data with multiple layers — like objects inside objects or lists of objects.

------------------------------------------------------------------------

## Nested Body

``` python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import List

app = FastAPI()

class Address(BaseModel):
    street: str
    city: str
    zip_code: str

class User(BaseModel):
    name: str
    age: int
    address: Address  # Nested model

@app.post("/users/")
def create_user(user: User):
    return user
```

### Example JSON Body:

``` json
{
  "name": "Hoa",
  "age": 30,
  "address": {
    "street": "123 Main St",
    "city": "Hanoi",
    "zip_code": "100000"
  }
}
```

------------------------------------------------------------------------

## Nested Lists of Models

``` python
class Item(BaseModel):
    name: str
    quantity: int

class Order(BaseModel):
    order_id: str
    items: List[Item]

@app.post("/orders/")
def create_order(order: Order):
    return order
```

### Example JSON Body:

``` json
{
  "order_id": "ORD123",
  "items": [
    { "name": "Book", "quantity": 2 },
    { "name": "Pen", "quantity": 5 }
  ]
}
```

------------------------------------------------------------------------

## Why Use Nested Models?

- **Clean structure** for complex data

- **Automatic validation** of nested fields

- **Better documentation** in Swagger UI

- **Reusability** of models across endpoints
