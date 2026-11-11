# Dependency Injection

``` mermaid
        graph TD
    Request[Incoming HTTP Request] --> FastAPI[FastAPI]
    FastAPI -->|inspect route signature| DG[Build Dependency Graph]
    DG -->|Depends get_db| DB[call get_database_connection]
    DG -->|Depends verify_token| Auth[call verify_token]
    DB -->|yield db| Inject[Inject into route handler]
    Auth -->|return user| Inject
    Inject --> Handler[Route Handler Function<br>reads_items db auth]
    Handler -->|response| FastAPI
    FastAPI -->|after response| Cleanup[cleanup finally blocks<br>db.close etc]
    style DG fill:#9cf,stroke:#333
    style Inject fill:#9f9,stroke:#333
    style Cleanup fill:#fc9,stroke:#333
```

## What `Depends` does

In FastAPI, Depends is a dependency injection system. It lets you declare that your endpoint (or another dependency) depends on a certain function — and FastAPI will automatically call that function, handle its logic, and pass the returned object to where it’s needed.

``` python
from fastapi import FastAPI, Depends

app = FastAPI()

# A dependency function
def get_database_connection():
    db = {"connection": "Connected to DB"}  # pretend this is a DB connection
    try:
        yield db   # yield allows cleanup later
    finally:
        db["connection"] = "Closed"

# An endpoint that depends on that object
@app.get("/items/")
def read_items(db=Depends(get_database_connection)):
    return {"db_status": db["connection"]}
```

What happens:

FastAPI sees `db=Depends(get_database_connection)`. It calls get_database_connection(). It injects the returned object (db) into your endpoint function. It handles cleanup if you used yield.

## Why it’s powerful

You can use dependencies to:

- Open/close database connections

- Verify authentication

- Parse headers or tokens

- Reuse logic across multiple routes Dependencies can depend on other dependencies — forming a tree of logic.

------------------------------------------------------------------------

## What happens under the hood

- **Step 1**: FastAPI inspects the route handler’s signature When you write:

``` python
def read_items(db=Depends(get_database_connection)):
```

FastAPI doesn’t immediately call `get_database_connection()`. Instead, it records that `db` depends on `get_database_connection`.

It internally builds a dependency graph (a tree of all dependencies).

- **Step 2**: At request time When a request hits /users:

FastAPI checks the route’s declared dependencies. For each parameter that is Depends(some_func):

- It calls that function (some_func()), resolving its own dependencies recursively.

- It `awaits` it if it’s `async`.

- It injects the return value into the route function as the parameter.

So this:

`db=Depends(get_database_connection)` is not a normal Python function call like `db`` ``=`` ``get_database_connection()` — it’s a declarative marker telling FastAPI:

“When handling a request, please call `get_database_connection()` first, and pass its result here.”

- **Step 3**: Dependency context management If your dependency uses `yield`, like:

``` python
def get_database_connection():
    db = {"connection": "Connected to DB"}  # pretend this is a DB connection
    try:
        yield db   # yield allows cleanup later
    finally:
        db["connection"] = "Closed"
```

then FastAPI:

- Calls it as a context manager

- Keeps track of cleanup (`finally`) after the response is sent

- It handles this using Starlette’s `request.state` + `async` context stack internally.

------------------------------------------------------------------------

``` python
# Introspect dependency graph
for route in app.routes:
    if hasattr(route, "dependant"):
        print(f"\nRoute: {route.path}")
        dependant = route.dependant
        print("Top-level dependency:", dependant.call)
        for sub in dependant.dependencies:
            print(" ├─ depends on:", sub.call)
            for sub2 in sub.dependencies:
                print(" │   └─ depends on:", sub2.call)
```
