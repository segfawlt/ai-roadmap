# Server-Sent Events (SSE)

Server-Sent Events enable **one-way streaming** from server to client over HTTP. Unlike WebSockets (bidirectional), SSE is simpler, works over standard HTTP, and is the standard pattern for streaming AI/LLM responses in 2026.

FastAPI added **native SSE support** in v0.135.0 (March 2026) with `EventSourceResponse` and `ServerSentEvent`.

## When to Use SSE

| Pattern | Use Case | Protocol |
|----|----|----|
| **SSE** | Server pushes updates to client (LLM streaming, live feeds, notifications) | HTTP (one-way) |
| **WebSocket** | Bidirectional real-time communication (chat, gaming, collaborative editing) | WebSocket (two-way) |
| **Polling** | Client periodically checks for updates (simple, low-frequency) | HTTP (client-initiated) |

**Rule of thumb:** If the client only needs to *receive* data, use SSE. If the client needs to *send and receive* simultaneously, use WebSocket.

------------------------------------------------------------------------

## Basic SSE Endpoint

``` python
from fastapi import FastAPI
from fastapi.responses import EventSourceResponse
from fastapi.sse import ServerSentEvent
import asyncio

app = FastAPI()


async def event_generator():
    """Yield events one at a time."""
    for i in range(10):
        yield ServerSentEvent(data=f"Message {i}", event="update", id=str(i))
        await asyncio.sleep(0.5)
    # Final event to signal completion
    yield ServerSentEvent(data="[DONE]", event="complete")


@app.get("/stream")
async def stream_events():
    return EventSourceResponse(event_generator())
```

**Client (JavaScript):**

``` javascript
const source = new EventSource("/stream");

source.addEventListener("update", (event) => {
  console.log("Received:", event.data);
});

source.addEventListener("complete", (event) => {
  console.log("Stream complete");
  source.close();
});

source.onerror = (error) => {
  console.error("SSE error:", error);
  source.close();
};
```

------------------------------------------------------------------------

## Streaming LLM Responses

The most common SSE use case in 2026 — streaming AI-generated text token by token:

``` python
from fastapi import FastAPI
from fastapi.responses import EventSourceResponse
from fastapi.sse import ServerSentEvent
from pydantic import BaseModel


app = FastAPI()


class ChatRequest(BaseModel):
    message: str
    conversation_id: str | None = None


async def generate_llm_response(message: str):
    """Simulate streaming LLM response (replace with real LLM call)."""
    # In production: call OpenAI, Anthropic, or local model with stream=True
    response_tokens = f"Based on your question about {message}, here is my answer.".split()
    for token in response_tokens:
        yield token + " "


@app.post("/chat/stream")
async def stream_chat(request: ChatRequest):
    async def event_generator():
        async for token in generate_llm_response(request.message):
            yield ServerSentEvent(data=token, event="token")
        yield ServerSentEvent(data="[DONE]", event="done")

    return EventSourceResponse(event_generator())
```

------------------------------------------------------------------------

## SSE with Pydantic Models

FastAPI’s native SSE support includes built-in Pydantic serialization:

``` python
from pydantic import BaseModel
from fastapi.sse import ServerSentEvent


class ProgressUpdate(BaseModel):
    step: int
    total: int
    message: str
    percentage: float


async def processing_pipeline():
    steps = ["Validating input", "Querying database", "Generating report", "Complete"]
    for i, step in enumerate(steps):
        yield ServerSentEvent(
            data=ProgressUpdate(
                step=i + 1,
                total=len(steps),
                message=step,
                percentage=(i + 1) / len(steps) * 100,
            ),
            event="progress",
        )
        await asyncio.sleep(1)
```

------------------------------------------------------------------------

## Error Handling and Reconnection

SSE has built-in reconnection — if the connection drops, the browser automatically reconnects:

``` python
@app.get("/stream/resilient")
async def resilient_stream():
    async def event_generator():
        try:
            for i in range(100):
                yield ServerSentEvent(
                    data=f"Event {i}",
                    id=str(i),  # Client sends Last-Event-ID on reconnect
                    retry=5000,  # Reconnect after 5 seconds if disconnected
                )
                await asyncio.sleep(1)
        except asyncio.CancelledError:
            # Client disconnected — clean up resources
            print("Client disconnected, cleaning up")
            raise

    return EventSourceResponse(event_generator())
```

| Field   | Purpose                                              |
|---------|------------------------------------------------------|
| `data`  | The event payload (string or Pydantic model)         |
| `event` | Event type name (client filters by this)             |
| `id`    | Event ID (sent back as `Last-Event-ID` on reconnect) |
| `retry` | Milliseconds before client auto-reconnects           |

------------------------------------------------------------------------

## SSE vs WebSocket Decision Guide

``` mermaid
        flowchart TD
    A[Need real-time data?] -->|Yes| B{Direction?}
    A -->|No| C[Use REST API]
    B -->|Server → Client only| D[Use SSE]
    B -->|Bidirectional| E[Use WebSocket]
    D --> F{Data type?}
    F -->|Text/JSON stream| G[SSE with EventSourceResponse]
    F -->|Binary data| H[Use WebSocket instead]
```

| Feature            | SSE                 | WebSocket             |
|--------------------|---------------------|-----------------------|
| Direction          | Server → Client     | Bidirectional         |
| Protocol           | HTTP                | WebSocket (ws://)     |
| Auto-reconnect     | Built-in            | Manual implementation |
| Browser support    | All modern browsers | All modern browsers   |
| Proxy/CDN friendly | Yes (standard HTTP) | Sometimes problematic |
| Binary data        | No (text only)      | Yes                   |
| Complexity         | Low                 | Medium                |

------------------------------------------------------------------------

## Summary

| Concept | Key Point |
|----|----|
| **SSE** | One-way server-to-client streaming over HTTP |
| **EventSourceResponse** | FastAPI’s native SSE response class (v0.135+) |
| **ServerSentEvent** | Structured event with data, event type, id, retry |
| **LLM streaming** | The primary use case — stream tokens as they are generated |
| **Reconnection** | Built-in via `id` field and `Last-Event-ID` header |
| **vs WebSocket** | Use SSE when client only receives; WebSocket when bidirectional |

## References

1.  [FastAPI SSE Documentation](https://fastapi.tiangolo.com/tutorial/server-sent-events/)

2.  [MDN — Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)

3.  [FastAPI v0.135.0 Release Notes](https://fastapi.tiangolo.com/release-notes/)
