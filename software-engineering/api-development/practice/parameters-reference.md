# main-memory-fastAPI.py

``` python
from typing import Union

from fastapi import FastAPI

from pydantic import BaseModel


import uvicorn # for custom run only

class Item(BaseModel):
    name: str
    price: float
    is_offer: Union[bool, None] = None



app = FastAPI()

_d = {}


@app.get("/")
async def read_root():
    return {"Hello": "World"}


@app.get("/items/{item_id}")
async def read_item(item_id: int, q: Union[str, None] = None):
    # return {"item_id": item_id, "q": q}
    return _d[item_id]


import json

class ItemEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, Item):
            return {
                'name': obj.name,
                'price': obj.price,
                'is_offer': obj.is_offer
            }
        return super().default(obj)


@app.get("/items-all")
async def get_all():
    return json.dumps(_d, cls=ItemEncoder)


from datetime import datetime
from copy import deepcopy

@app.put("/items/{item_id}")
async def update_item(item_id, item: Item):
    # Do something with backend... then return
    # return {"item_name": item.name, "item_id": item_id, "updated_date": datetime.now()}
    if item_id not in _d:
        raise Exception(f"Item Id: {item_id} is not found")

    updated_item = deepcopy(item)
    updated_item.updated_datetine = datetime.now()
    _d[item_id] = updated_item
    return updated_item

@app.post("/item")
async def add_item(item: Item):
    # Do something with backend... then return
    # return {"item_name": item.name, "item_id": item_id, "updated_date": datetime.now()}
    item_id = len(_d) + 1
    _d[item_id] = item

    return _d[item_id]


# Test Header
from fastapi import Header
@app.get("/item-header")
def read_items(user_agent: str = Header(None)):

    print("AAA", user_agent)
    return {"User-Agent": user_agent}

@app.get("/item-header2")
def read_items(x_token: str = Header(...)):

    return {"X-Ttoken": x_token}

if __name__ == "__main__":
    uvicorn.run(app, host="127.0.0.1", port=8000)
```
