# Assignment: Building a Product Inventory API with FastAPI and Pydantic

**Objective:** Implement a FastAPI service that demonstrates proficiency in defining routes, handling different parameter types (path and query), enforcing data validation using Pydantic, and managing complex request bodies and responses.

## Section 1: Route Definition and Path Parameters (Sources)

**Task 1.1: Basic and Validated Path Parameters (15 Points)**

1.  Create a FastAPI application instance.

2.  Define a `GET` endpoint for `/products/{product_id}`.

    - The `product_id` must be captured as a **path parameter**.

    - It must be automatically converted to an `integer`.

    - Implement **validation** using `Path()` to ensure the `product_id` is **greater than zero (gt=0)** and is required (`...`).

    - The function should return the product ID in JSON format.

    - *Hint: Refer to the mapping table pattern provided in the sources*.

**Task 1.2: Capturing Paths (10 Points)**

1.  Define a `GET` endpoint for `/docs/{file_path:path}`.

2.  Use the **`:path` annotation** in the URL pattern to ensure the `file_path` parameter captures the entire path, including forward slashes (`/`).

3.  The function should return the captured file path.

## Section 2: Query Parameters and Validation

**Task 2.1: Default and Optional Query Parameters (15 Points)**

1.  Define a `GET` endpoint for `/inventory/`.

2.  Implement two query parameters for pagination, following the basic example pattern:

    - `skip`: An integer that defaults to `0`.

    - `limit`: An integer that defaults to `10`.

3.  Implement an optional boolean query parameter named `in_stock`.

    - This parameter should default to `False`.

    - Demonstrate that FastAPI handles boolean type conversion (e.g., accepts “1”, “true”, or “yes”).

**Task 2.2: Required Query Parameters and Advanced Validation (15 Points)**

1.  Define a `GET` endpoint for `/search/`.

2.  The endpoint must accept a required query parameter named `keyword`.

3.  Use Pydantic’s `Query()` function to enforce **string length and pattern validation** on `keyword`:

    - Minimum length must be `3`.

    - Maximum length must be `50`.

4.  Add a numerical query parameter `min_rating` and use `Query()` to enforce **range validation**:

    - The rating must be **greater than or equal to 0 (`ge=0`)**.

## Section 3: Data Modeling, Request Body, and Validation

**Task 3.1: Pydantic Data Model for Request Body (15 Points)**

1.  Define a Pydantic `BaseModel` named `Supplier`.

2.  The `Supplier` model must contain:

    - `name` (string).

    - `rating` (integer, must be **greater than 0**).

    - `is_primary` (boolean, defaults to `True`).

3.  Create a `POST` endpoint `/suppliers/` that accepts the `Supplier` object in the **JSON request body**.

4.  Demonstrate that FastAPI automatically validates the request body against the Pydantic model and returns a 422 error if validation fails.

**Task 3.2: Nested Models (10 Points)**

1.  Create a system for processing bulk orders by defining **nested Pydantic models**:

    - Model `OrderItem`: fields `product_name`, `price`, `quantity`.

    - Model `CustomerInfo`: fields `name`, `email`.

    - Model `BulkOrder`: must contain `customer` (which uses `CustomerInfo`) and a list of items (`items:`` ``List[OrderItem]`).

2.  Create a `POST` endpoint `/orders/` that accepts the `BulkOrder` model in the request body.

## Section 4: Output Filtering

**Task 4.1: Securing Output with `response_model` (20 Points)**

You are registering new system administrators. You must accept a password but never return it in the response.

1.  Define two Pydantic models:

    - `AdminIn`: Includes `username` and `password`.

    - `AdminOut`: Includes only `username`.

2.  Create a `POST` endpoint `/admin/register`.

3.  The endpoint should accept the complete `AdminIn` model in the request body.

4.  Use the **`response_model`** parameter in the route decorator to ensure that the function’s return value is filtered to match the structure defined in `AdminOut`.

5.  Prove that even if the function internally returns the full input (including the password), the API response only contains the fields specified by `AdminOut`.

------------------------------------------------------------------------

## Summary of Key Concepts Tested

This assignment requires students to utilize the following core concepts covered in the sources:

- **Path vs. Query:** Distinguishing parameter locations.

- **Automatic Type Conversion and Validation:** Using Python type hints (e.g., `int`).

- **Advanced Validation:** Applying Pydantic functions (`Path()`, `Query()`) for constraints like `gt=0`, `min_length`, and `ge`.

- **Required vs. Optional Parameters:** Handling parameters with and without default values (`=`` ``None` or no default).

- **Data Modeling:** Structuring complex inputs using `BaseModel`.

- **Complex Payloads:** Handling hierarchical data using **Nested Pydantic Models** in the request body.

- **Security/Filtering:** Controlling the structure of the API output using the `response_model`.

**Analogy for Pydantic Models:** Think of a Pydantic model for a request body like a **pre-flight customs form** for data. When data (the traveler) arrives at your API (the border), it must fill out the form (the Pydantic model). If the data doesn’t meet the requirements (wrong field types, missing required information, or values outside acceptable ranges like a password being too short), FastAPI (the customs agent) automatically rejects it with an error (422) before it even reaches your main processing logic.
