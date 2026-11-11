# Async & Event Loop Exploration

These code snippets explore how Python’s asyncio event loop works and how FastAPI runs on top of it. Run them in a Jupyter notebook to observe the behavior.

## Event Loop Policy Inspection

Examine the default event loop policy and its internal state:

``` python
import asyncio

# asyncio.set_event_loop_policy(asyncio.unix_events.DefaultEventLoopPolicy())


test = asyncio.get_event_loop_policy()
```

``` python
test._hai_thread
```

``` python
# dir(test)

test._hai_stack
```

``` python
```

## Running FastAPI in a Background Thread

This demonstrates how Uvicorn can run FastAPI in a daemon thread — useful for Jupyter notebook experimentation where you want to test endpoints interactively:

``` python
import uvicorn
from fastapi import FastAPI

import threading


app = FastAPI()

@app.get("/")
def read_root():
    return {"hello": "world"}

def run_app():
    uvicorn.run(app, host="0.0.0.0", port=8888)

thread = threading.Thread(target=run_app, daemon=True, name="Thread run FASTAPI")
thread.start()
```

```
INFO:     Started server process [35520]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8888 (Press CTRL+C to quit)


INFO:     127.0.0.1:55111 - "GET / HTTP/1.1" 200 OK
INFO:     127.0.0.1:55143 - "GET / HTTP/1.1" 200 OK
```

``` python
import sys
```

``` python
sys.modules
```

``` python
import asyncio

test = asyncio.events._event_loop_policy
```

``` python
test._local
```

```
<asyncio.events.BaseDefaultEventLoopPolicy._Local at 0x10320fef0>
```

``` python
id(test._local._loop)
```

## Advanced: Custom Event Loop Tracking

Internal debugging technique for tracking event loop creation (instructor reference):

``` python
# HoHai: Debug
# ensure the class has the attribute, then append
lst = getattr(self.__class__, "_hai_loop", None)
if lst is None:
    setattr(self.__class__, "_hai_loop", [self])
else:
    lst.append(self)
```
