# What ASGI Is

**ASGI** is the spiritual successor to **WSGI** (Web Server Gateway Interface).

- **WSGI** was made for **synchronous** Python web apps (e.g. Flask, Django before Channels).

- **ASGI** extends this idea to support **asynchronous** communication — like **WebSockets**, **HTTP/2**, and **background tasks**.

In simple terms:

> ASGI defines how an async Python web app communicates with an async web server (like Uvicorn, Hypercorn, Daphne).

## What an `ASGIApplication` Is

An `ASGIApplication` is just **any callable object that follows the ASGI specification**, i.e.:

``` python
async def app(scope, receive, send):
    ...
```

- **scope**: Information about the connection (type = “http”, “websocket”, etc.)

- **receive**: An awaitable that yields incoming events (like HTTP requests, WebSocket messages)

- **send**: A callable used to send responses/events back to the client

So, a framework might define a class like:

``` python
class ASGIApplication:
    async def __call__(self, scope, receive, send):
        # handle the request
        ...
```

This makes it a **valid ASGI app**, because any ASGI server can call it.

## Why Frameworks Use the Name `ASGIApplication`

Frameworks often call something `ASGIApplication` because:

1.  **It’s the entry point** that the ASGI server will invoke (similar to `WSGIApplication` in older frameworks).

2.  **It clearly communicates compatibility** — the framework or component is designed to be used in an ASGI environment.

3.  **It’s flexible** — multiple ASGI apps can be composed together (e.g. mounting FastAPI inside Starlette).

## Example: Minimal Custom ASGI Application

``` python
class SimpleASGIApp:
    async def __call__(self, scope, receive, send):
        if scope['type'] == 'http':
            await send({
                'type': 'http.response.start',
                'status': 200,
                'headers': [(b'content-type', b'text/plain')],
            })
            await send({
                'type': 'http.response.body',
                'body': b'Hello, ASGI world!',
            })
```

You could then run this with:

``` bash
uvicorn myapp:SimpleASGIApp
```

## Summary

| Concept | Description |
|----|----|
| **ASGI** | Async Server Gateway Interface (modern, async version of WSGI) |
| **ASGIApplication** | A callable object conforming to ASGI spec (`__call__(scope,`` ``receive,`` ``send)`) |
| **Why the name?** | To indicate that the object *is* an ASGI-compatible application entry point |

## ASGI in FastAPI

### Overview diagram

``` mermaid
        graph TD
    Client[HTTP Client] -->|TCP connection| Uvicorn[Uvicorn Server<br>asyncio event loop]
    Uvicorn -->|H11 HTTP/1.1 protocol| H11[H11 Protocol Handler]
    H11 -->|ASGI call __call__<br>scope receive send| FastAPI[FastAPI app]
    FastAPI -->|inspect URL + method| Router[Router / Routes]
    Router -->|matched route| PathOp[Path Operation Function]
    PathOp -->|response| FastAPI
    FastAPI -->|send response events| Uvicorn
    Uvicorn -->|HTTP response| Client
    style Uvicorn fill:#9cf,stroke:#333
    style FastAPI fill:#9f9,stroke:#333
    style Router fill:#fc9,stroke:#333
```

### Key components

**Uvicorn Server:** Acts as the high-performance ASGI server that handles low-level network communication. It runs an asynchronous event loop using `asyncio.get_running_loop`.

**FastAPI `app`**: The Python application framework that defines the API logic.

**H11 Protocol**: The default implementation in Uvicorn for handling the HTTP/1.1 protocol.

**Router/Routes**: Mechanisms within FastAPI (built on Starlette) that map the incoming request URL and HTTP method to the appropriate path operation function.

### Request Flow overview

1.  **Incoming Request**: A request arrives at the Uvicorn server’s listening host and port.

2.  **Protocol Handling**: The Uvicorn server, via the H11 protocol implementation, processes the raw incoming HTTP request data.

3.  **ASGI Interface**: The server communicates with the FastAPI application using the ASGI standard. It calls the application’s `__call__` method, passing specific `scope`, `receive`, and `send` channels to manage the request’s lifecycle asynchronously.

4.  **Routing**: FastAPI’s internal router inspects the request details (URL path and HTTP method) and matches it to a defined route.

5.  **Path Operation Function**: Once a match is found, the associated path operation function is executed to process the request logic and generate a response.
