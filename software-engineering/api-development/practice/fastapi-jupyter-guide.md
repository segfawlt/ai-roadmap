# Try to run `FastAPI` in Jupyter Notebook

- For study only (not for production)

- Need to install `nest-asyncio`

``` sh
!pip install fastapi uvicorn nest-asyncio
```

``` bash
!pip install fastapi uvicorn nest-asyncio
```

``` python
import uvicorn
from fastapi import FastAPI
import nest_asyncio
import threading

nest_asyncio.apply()   # Allows nested event loops (required in notebooks)

app = FastAPI()

@app.get("/")
async def read_root():
    return {"hello": "world"}

def run_app():
    uvicorn.run(app, host="0.0.0.0", port=8000)

thread = threading.Thread(target=run_app, daemon=True, name="Thread run FASTAPI")
thread.start()
```

```
INFO:     Started server process [39889]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

Uvicorn, when started this way, creates one asyncio event loop bound to the thread it runs in.

- Each event loop must run in a single thread.

- Uvicorn will not create multiple loops unless you explicitly run multiple servers or workers.

``` python
```
