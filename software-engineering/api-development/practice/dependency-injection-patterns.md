# Diagram

``` mermaid
        graph TD
    App["Top-level App<br>(ASGI application)"] -- register --> Router["API Router"]
    Router -- maps path e.g. '/users' --> Endpoint["Endpoint<br>(callable: read_users)"]
    Endpoint -- declares inputs --> Inputs["p1: str, p2: str, ..."]
    Endpoint -- declares --> Deps["db: Depends(fn1)<br>...Depends(fn2)"]
    Deps -- resolved by --> DI["Dependency Injection<br>(Depends)"]
    DI -- injects --> Fn["Function / Callable Object"]
    Fn -- returns response --> Endpoint
```

**Routing in Starlette:** The process starts with a top-level app (which is an ASGI application). This app registers Routes, often organized within an API Router. A route maps a specific URL path (e.g., “/users”) to an endpoint, which is a function or callable object (like read_users).

**Dependency Injection (DI):**

This design pattern allows functions or objects to declare what they need to work, which are then “injected” into them, rather than the function creating those dependencies itself.

**Depends Function:** The core mechanism for DI is the Depends() function. It is used within endpoint function signatures to define dependencies (e.g., db: Depends(get_db)). These dependencies can be complex logic, database sessions, or configuration settings.

**How it Works Together:**

1.  When a request matches a route, the system inspects the signature of the endpoint function (e.g., `read_users(p1:`` ``str,`` ``db:`` ``Depends(...))`).

2.  It identifies the required inputs (p1, db).

3.  It resolves the dependencies defined by Depends() (represented by fn1, fn2, etc.).

4.  These resolved dependencies are then passed to the endpoint function as arguments.

5.  The endpoint function executes with all its necessary components provided, improving modularity and testability.
