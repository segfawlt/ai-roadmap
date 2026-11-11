# FastAPI Code Examples for Query Parameters

This page provides annotated code examples for working with query parameters in FastAPI, including default values, optional parameters, type validation, and grouping parameters into Pydantic models.

## 1. Basic Query Parameters and Defaults

If you assign a default value to a parameter, that parameter is not required. FastAPI will use the default value if the query parameter is not provided in the URL.

In the example below, `skip` defaults to `0` and `limit` defaults to `10`.

| Example URL                            | Resulting Function Values |
|----------------------------------------|---------------------------|
| `http://127.0.0.1:8000/items/`         | `skip=0`, `limit=10`      |
| `http://127.0.0.1:8000/items/?skip=20` | `skip=20`, `limit=10`     |

``` python
from fastapi import FastAPI

app = FastAPI()

fake_items_db = [{"item_name": "Foo"}, {"item_name": "Bar"}, {"item_name": "Baz"}]

@app.get("/items/")
async def read_item(skip: int = 0, limit: int = 10):
    return fake_items_db[skip : skip + limit]
```

## 2. Optional Query Parameters

To make a query parameter optional without assigning a specific default value, set its default to `None`.

``` python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: str, q: str | None = None):
    if q:
        return {"item_id": item_id, "q": q}
    return {"item_id": item_id}
```

## 3. Boolean Type Conversion

FastAPI automatically converts incoming query parameter strings into the declared Python type. For boolean types (`bool`), FastAPI accepts several values as `True`, including `1`, `True`, `true`, `on`, or `yes`.

``` python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: str, q: str | None = None, short: bool = False):
    item = {"item_id": item_id}
    if q:
        item.update({"q": q})
    if not short:
        item.update(
            {"description": "This is an amazing item that has a long description"}
        )
    return item
```

## 4. Required Query Parameters

To declare a query parameter as **required**, declare the parameter without assigning any default value.

| Example URL | Result |
|----|----|
| `http://127.0.0.1:8000/items/foo-item` | Error: `Field`` ``required` for `needy` |
| `http://127.0.0.1:8000/items/foo-item?needy=sooooneedy` | Success |

``` python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
async def read_user_item(item_id: str, needy: str):
    item = {"item_id": item_id, "needy": needy}
    return item
```

## 5. Mixing Path, Required, Default, and Optional Parameters

You can define multiple parameter types simultaneously, and FastAPI will automatically determine which are path parameters and which are query parameters.

``` python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
async def read_user_item(
    item_id: str,
    needy: str,
    skip: int = 0,
    limit: int | None = None
):
    item = {"item_id": item_id, "needy": needy, "skip": skip, "limit": limit}
    return item
```

## Query Parameter Validations

In FastAPI, validation in query parameters refers to the **automatic checking** and **enforcement of rules** on the values passed through the URL’s query string. FastAPI uses Pydantic and type hints to perform this validation.

### How Validation Works

FastAPI validates query parameters using:

- Type annotations (e.g., `int`, `str`, `bool`)

- Default values

- Pydantic’s `Query()` function for more advanced validation

If a user passes `?skip=abc`, FastAPI will return a `422`` ``Unprocessable`` ``Entity` error because `abc` is not an integer.

### Advanced Validation with Query

``` python
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
def read_items(
    q: str = Query(..., min_length=3, max_length=50, regex="^item")
):
    return {"q": q}
```

### Common Validation Examples

**1. Basic Type Validation**

``` python
@app.get("/products/")
def get_products(page: int = 1, size: int = 10):
    return {"page": page, "size": size}
```

**2. String Length and Pattern Validation**

``` python
@app.get("/search/")
def search_items(
    keyword: str = Query(..., min_length=3, max_length=50, regex="^[a-zA-Z0-9 ]+$")
):
    return {"keyword": keyword}
```

- `...` makes `keyword` **required**

- Enforces minimum length, maximum length, and character pattern

**3. Optional Parameters with Default Values**

``` python
@app.get("/filter/")
def filter_items(
    category: str = Query(None),
    in_stock: bool = Query(False)
):
    return {"category": category, "in_stock": in_stock}
```

**4. List of Values**

``` python
@app.get("/tags/")
def get_items_by_tags(tags: list[str] = Query([])):
    return {"tags": tags}
```

Accepts: `/tags/?tags=python&tags=fastapi&tags=api`

**5. Range Validation for Numbers**

``` python
@app.get("/ratings/")
def get_ratings(
    min_rating: float = Query(0, ge=0, le=5),
    max_rating: float = Query(5, ge=0, le=5)
):
    return {"min_rating": min_rating, "max_rating": max_rating}
```

`ge` = greater than or equal, `le` = less than or equal

**6. Using Aliases for Query Parameters**

``` python
@app.get("/items/")
def get_items(
    sort_by: str = Query("name", alias="sortBy")
):
    return {"sort_by": sort_by}
```

Accepts `?sortBy=price` in the URL, maps to `sort_by` in the function.

**7. Deprecating a Parameter**

``` python
@app.get("/legacy/")
def legacy_endpoint(
    old_param: str = Query(None, deprecated=True)
):
    return {"old_param": old_param}
```

Marks `old_param` as deprecated in the OpenAPI docs.

## Query Parameter with Pydantic Models

A query parameter model groups and validates multiple query parameters using a Pydantic model. This is useful when you have many related query parameters and want to keep endpoint functions clean and organized.

### Why Use Query Parameter Models?

- Keeps code cleaner and more maintainable

- Enables reusability of validation logic

- Automatically generates OpenAPI documentation

- Supports default values, type checking, and validation rules

**Without a Model:**

``` python
from fastapi import FastAPI
app = FastAPI()

@app.get("/items/")
def get_items(skip: int = 0, limit: int = 10, search: str = None):
    return {"skip": skip, "limit": limit, "search": search}
```

**With a Query Parameter Model:**

``` python
from fastapi import FastAPI, Depends
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class ItemQueryParams(BaseModel):
    skip: int = 0
    limit: int = 10
    search: Optional[str] = None

@app.get("/items/")
def get_items(params: ItemQueryParams = Depends()):
    return params
```

`Depends()` tells FastAPI to extract query parameters and validate them using the `ItemQueryParams` model. FastAPI will automatically parse query parameters, validate types, apply default values, and return a 422 error if validation fails.
