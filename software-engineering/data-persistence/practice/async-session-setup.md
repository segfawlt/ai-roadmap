# Becareful

``` python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker


# AsyncSessionLocal = sessionmaker(
#     engine, expire_on_commit=False, class_=AsyncSession
# )

# async def get_db_connection():
#     db = AsyncSessionLocal() # Create an instance for every calling time
#     try:
#         yield db
#     finally:
#         await db.close()

AsyncSessionLocal = async_sessionmaker(
    engine, expire_on_commit=False, class_=AsyncSession
)

async def get_db_connection():
    async with AsyncSessionLocal() as db:
        yield db
```
