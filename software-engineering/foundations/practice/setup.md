``` bash
!pip install "fastapi[standard]"
```

# Write your first API apps

``` python
from typing import Union

from fastapi import FastAPI

from pydantic import BaseModel


class Item(BaseModel):
    name: str
    price: float
    is_offer: Union[bool, None] = None


app = FastAPI()


@app.get("/")
async def read_root():
    return {"Hello": "World"}


@app.get("/items/{item_id}")
async def read_item(item_id: int, q: Union[str, None] = None):
    return {"item_id": item_id, "q": q}


from datetime import datetime

@app.put("/items/{item_id}")
async def update_item(item_id, item: Item):
    # Do something with backend... then return
    return {"item_name": item.name, "item_id": item_id, "updated_date": datetime.now()}
```

## Run App

- with default command of `fastapi`

``` sh
fastapi dev main.py
```

## Debug App

- Host with `uvicorn`

``` python
import uvicorn # Import `uvicorn`  lightweight ASGI server
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def root():
    a = "a"
    b = "b" + a
    return {"hello world": b}


if __name__ == "__main__":
    uvicorn.run(app, host="127.0.0.1", port=8000) # Run app with `uvicorn`
```

![VS Code debugger launch configuration setup](https://ai-roadmap.ops-ai.dev/_images/2025-10-21-15-28-37.png)

Then VS Code automatically creates a `tasks.json` file to run as below:

``` json
{
  // Use IntelliSense to learn about possible attributes.
  // Hover to view descriptions of existing attributes.
  // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python Debugger: Current File",
      "type": "debugpy",
      "request": "launch",
      "program": "${file}",
      "console": "integratedTerminal"
    }
  ]
}
```

- Now can press F5 to start program with debugger.

- Breakpoints are encountered as you place them

![VS Code debugger stopped at breakpoint](https://ai-roadmap.ops-ai.dev/_images/2025-10-21-15-30-42.png)
