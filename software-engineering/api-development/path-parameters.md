# What are Path Parameters in FastAPI?

**Path parameters** are variables that are part of the URL path. They allow you to capture values from the URL and use them inside your function.

For example, in the URL:

```
/items/42
```

`42` can be captured as a path parameter.

## Basic Example

``` python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}
```

### Explanation:

- `@app.get("/items/{item_id}")`: This defines a route where `item_id` is a path parameter.

- `item_id:`` ``int`: FastAPI automatically converts the path parameter to an integer.

- The function returns the value of `item_id` as JSON.

## Multiple Path Parameters

``` python
@app.get("/users/{user_id}/items/{item_id}")
async def read_user_item(user_id: int, item_id: str):
    return {"user_id": user_id, "item_id": item_id}
```

You can define multiple parameters in the path and FastAPI will parse them accordingly.

## Path Parameter with Validation

You can also use **Pydantic** and **Path** for validation:

``` python
from fastapi import Path

@app.get("/products/{product_id}")
async def get_product(product_id: int = Path(..., gt=0, description="The ID must be greater than 0")):
    return {"product_id": product_id}
```

- `gt=0`: Ensures the ID is greater than 0.

- `...`: Means the parameter is required.

## Path Parameters vs Query Parameters

- **Path parameters** are part of the URL path: `/items/42`

- **Query parameters** are after the `?`: `/items?skip=0&limit=10`

Here’s a **table pattern** that shows how **FastAPI path parameters** map URLs to functions and parameters. This can help you visualize how different URL structures correspond to function calls.

### FastAPI Path Parameter Mapping Table

| **URL Pattern** | **Function Signature** | **Path Parameters** | **Example URL** | **Example Output** |
|----|----|----|----|----|
| `/items/{item_id}` | `def`` ``read_item(item_id:`` ``int)` | `item_id:`` ``int` | `/items/42` | `{`` ``"item_id":`` ``42`` ``}` |
| `/users/{user_id}/orders/{order_id}` | `def`` ``get_order(user_id:`` ``int,`` ``order_id:`` ``str)` | `user_id:`` ``int`, `order_id:`` ``str` | `/users/7/orders/abc123` | `{`` ``"user_id":`` ``7,`` ``"order_id":`` ``"abc123"`` ``}` |
| `/files/{file_path:path}` | `def`` ``read_file(file_path:`` ``str)` | `file_path:`` ``str` (captures slashes) | `/files/docs/readme.txt` | `{`` ``"file_path":`` ``"docs/readme.txt"`` ``}` |
| `/products/{product_id}` | `def`` ``get_product(product_id:`` ``int`` ``=`` ``Path(...,`` ``gt=0))` | `product_id:`` ``int` with validation | `/products/10` | `{`` ``"product_id":`` ``10`` ``}` |
| `/blog/{year}/{month}/{slug}` | `def`` ``read_blog(year:`` ``int,`` ``month:`` ``int,`` ``slug:`` ``str)` | `year`, `month`, `slug` | `/blog/2025/10/fastapi-path` | `{`` ``"year":`` ``2025,`` ``"month":`` ``10,`` ``"slug":`` ``"fastapi-path"`` ``}` |

### Notes:

- Use `:path` to capture the entire path including slashes.

- Use `Path()` from `fastapi` to add validation (e.g., `gt=0`, `min_length=3`).

- FastAPI automatically converts types based on annotations.
