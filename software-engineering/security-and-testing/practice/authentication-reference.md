# **init**.py

```
```## main.py
```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session


from models import schemas


from db_core.db import get_db_connection

import uvicorn

from models import crud

# Create tables
# Remove (refer to init_models)
# Base.metadata.create_all(bind=engine)


# expose auth API
from auth.auth import get_password_hash, auth_server
from fastapi.security import OAuth2PasswordRequestForm, OAuth2PasswordBearer
from fastapi import status


# from fastapi.security import OAuth2PasswordBearer
# from fastapi import Depends, HTTPException, status

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

app = FastAPI()

from starlette.middleware.trustedhost import TrustedHostMiddleware

app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["*", "*.devtunnels.ms"]
)



@app.post("/users/", response_model=schemas.UserRead)
async def create_user(user: schemas.UserCreate, db: Session = Depends(get_db_connection)):
    db_user = await crud.create_user(db=db,
                                     user=schemas.UserCreate(
                                         name=user.name,
                                         email=user.email,
                                         password=get_password_hash(user.password)))


    return db_user

@app.get("/users/")
async def read_users(skip: int = 0, limit: int = 10, db: Session = Depends(get_db_connection)):
    users = await crud.get_users(db, skip=skip, limit=limit)

    return users

@app.get("/users/{username}", response_model=schemas.UserRead)
async def read_user(username: str, db: Session = Depends(get_db_connection)):
    db_user = await crud.get_user_by_username(db, username==username)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user

# Bad behavior
@app.post("/users/{username}")
async def update_user(username: str, user: schemas.UserCreate, db: Session = Depends(get_db_connection)):
    db_user = await crud.update_user(db, username=username, user=user)
    return db_user

@app.delete("/users/{username}")
async def delete_user(username: str, db = Depends(get_db_connection)):
    db_user = await crud.delete_user(db, username)
    if not db_user:
        raise HTTPException(status_code=404, detail="User not found")

    return db_user



@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    token_object = await auth_server.issue_token(form_data.username, form_data.password)
    if token_object is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    return token_object


# from auth.auth_in_fastapi import oauth2_scheme, create_post_with_auth
from models.crud import create_post

@app.post("/post")
async def create_post_with_protected(content: str, db: Session = Depends(get_db_connection), token: str = Depends(oauth2_scheme)):
    payload = auth_server.verify_token(token) # throw exception if not

    post = await create_post(db, content)
    # result = await create_post_with_auth(content=content, db=db, token=token)
    return post



if __name__ == "__main__":
    # TODO: Should be run here to make sure that all tables (which mapped with models) will be scan in `metadata`
    # asyncio.run(init_models())



    # uvicorn.run(app, host="127.0.0.1", port=8000, reload=True)
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
    # uvicorn.run("main:app", host="127.0.0.1", port=8000)
```## auth.py
```python
# Build AuthServer
from datetime import datetime, timedelta
from jose import JWTError, jwt
import inspect
from passlib.context import CryptContext

# Need to make async :(
class AuthServerAsync:
    def __init__(self):
        # You can have more flexibilites by putting these info in contructor
        self._access_token_expire_mins = 30
        self._secret_key = "supersecretkey123"
        self._algorithm = "HS256"

    async def authen_user(self, *args, **kwargs) -> dict:
        raise NotImplementedError()

    async def issue_token(self, *args, **kwargs):
        user_dict = await self.authen_user(*args, **kwargs)
        if user_dict is None:
            raise Exception("User not authenticated")

        to_encode = user_dict.copy()
        expire = datetime.now() + timedelta(minutes=self._access_token_expire_mins)
        to_encode.update({"exp": expire})
        return jwt.encode(to_encode, self._secret_key, algorithm=self._algorithm)

    async def verify_token(self, token: str) -> dict:
        try:
            payload = jwt.decode(token, self._secret_key, algorithms=[self._algorithm])
            return payload
        except JWTError as e:
            raise e



class AuthUserPassServerAsync(AuthServerAsync):
    def __init__(self, fn_check_userpass):
        super().__init__()
        assert fn_check_userpass is not None, "Must set `fn_check_userpass`"
        self.fn = fn_check_userpass

    # Overriding sync with async breaks Liskov Substitution
    async def authen_user(self, username, password) -> dict:
        # Support async functions
        # if inspect.iscoroutinefunction(self.fn):
        #     import asyncio
        #     import nest_asyncio

        #     # Support jupyter
        #     nest_asyncio.apply()

        #     loop = asyncio.get_event_loop()
        #     task = loop.create_task(self.fn(username, password))
        #     loop.run_until_complete(task)
        #     return task.result()
        # else:
        #     return self.fn(username, password)

        if not inspect.iscoroutinefunction(self.fn):
            raise Exception("Please make sure you put `sync` function")

        return await self.fn(username, password)

class UnAuthorizedException(Exception):
    def __init__(self, *args):
        super().__init__("UnAuthorized.")





import asyncio
from greenlet import greenlet

async def greenlet_spawn(func, *args, **kwargs):
    loop = asyncio.get_running_loop()
    fut = loop.create_future()

    def run():
        try:
            res = func(*args, **kwargs)
            loop.call_soon_threadsafe(fut.set_result, res)
        except BaseException as e:
            loop.call_soon_threadsafe(fut.set_exception, e)

    g = greenlet(run)
    g.switch()
    return await fut

# class AuthUserPassServerAsync:
#     def __init__(self, sync_server: AuthUserPassServer):
#         self.sync_server = sync_server

#     async def authen_user(self, username, password):
#         fn = self.sync_server.fn

#         # Case A: async function provided
#         if inspect.iscoroutinefunction(fn):
#             return await fn(username, password)

#         # Case B: sync function provided → offload to greenlet
#         return await greenlet_spawn(fn, username, password)

#     async def issue_token(self, *args, **kwargs):
#         return await greenlet_spawn(self.sync_server.issue_token, *args, **kwargs)

#     async def verify_token(self, token):
#         return await greenlet_spawn(self.sync_server.verify_token, token)

# import asyncio
# from greenlet import greenlet

# async def greenlet_spawn(func, *args, **kwargs):
#     loop = asyncio.get_running_loop()
#     fut = loop.create_future()

#     def run():
#         try:
#             res = func(*args, **kwargs)
#             loop.call_soon_threadsafe(fut.set_result, res)
#         except BaseException as e:
#             loop.call_soon_threadsafe(fut.set_exception, e)

#     greenlet(run).switch()
#     return await fut


# class AuthUserPassServerAsync:
#     def __init__(self, sync_server: AuthUserPassServer):
#         self.sync = sync_server

#     # -------------------------------------------
#     # ASYNC user authentication
#     # -------------------------------------------
#     async def authen_user(self, username, password):
#         fn = self.sync.fn

#         if inspect.iscoroutinefunction(fn):
#             return await fn(username, password)
#         else:
#             return await greenlet_spawn(fn, username, password)

#     # -------------------------------------------
#     # ASYNC token creation (reimplemented)
#     # -------------------------------------------
#     async def issue_token(self, username, password):
#         """Async version independent of sync logic."""
#         user_dict = await self.authen_user(username, password)

#         if not user_dict:
#             raise Exception("User not authenticated")

#         to_encode = user_dict.copy()
#         expire = datetime.now() + timedelta(
#             minutes=self.sync._access_token_expire_mins
#         )
#         to_encode.update({"exp": expire})

#         return jwt.encode(
#             to_encode,
#             self.sync._secret_key,
#             algorithm=self.sync._algorithm
#         )

#     # -------------------------------------------
#     # VERIFY token (still sync → run in greenlet)
#     # -------------------------------------------
#     async def verify_token(self, token):
#         return await greenlet_spawn(self.sync.verify_token, token)





pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    truncated = password.encode("utf-8")[:72].decode("utf-8", errors="ignore")
    return pwd_context.hash(truncated)

# Specified in FastAPI and our application
from models.crud import get_user_by_username
from db_core.db import get_db_connection

async def fn_check_user(username, password):
    # db = await get_db_connection()

    gen = get_db_connection()
    db = await gen.__anext__()  # get the yielded db

    try:

        user_in_db = await get_user_by_username(db, username)

        if not user_in_db or not verify_password(password, user_in_db.password):
            raise UnAuthorizedException()
        return {
            "username": "OK",
            "email": "Yes"
        }
    finally:
        await gen.aclose()

auth_server = AuthUserPassServerAsync(fn_check_user)

```## [TOBE-REMOVE]auth-bk.py
```python
# Build AuthServer
from datetime import datetime, timedelta
from jose import JWTError, jwt
import inspect
from passlib.context import CryptContext

class AuthServer:
    def __init__(self):
        # You can have more flexibilites by putting these info in contructor
        self._access_token_expire_mins = 30
        self._secret_key = "supersecretkey123"
        self._algorithm = "HS256"

    def authen_user(self, *args, **kwargs) -> dict:
        raise NotImplementedError()

    def issue_token(self, *args, **kwargs):
        user_dict = self.authen_user(*args, **kwargs)
        if user_dict is None:
            raise Exception("User not authenticated")

        to_encode = user_dict.copy()
        expire = datetime.now() + timedelta(minutes=self._access_token_expire_mins)
        to_encode.update({"exp": expire})
        return jwt.encode(to_encode, self._secret_key, algorithm=self._algorithm)

    def verify_token(self, token: str) -> dict:
        return jwt.decode(token, self._secret_key, algorithms=[self._algorithm])


class AuthUserPassServer(AuthServer):
    def __init__(self, fn_check_userpass):
        super().__init__()
        assert fn_check_userpass is not None, "Must set `fn_check_userpass`"
        self.fn = fn_check_userpass

    # Overriding sync with async breaks Liskov Substitution
    def authen_user(self, username, password) -> dict:
        # Support async functions
        if inspect.iscoroutinefunction(self.fn):
            # import asyncio
            # import nest_asyncio

            # # Support jupyter
            # nest_asyncio.apply()

            # loop = asyncio.get_event_loop()
            # task = loop.create_task(self.fn(username, password))
            # loop.run_until_complete(task)
            # return task.result()


            # Delegate async fn to the async wrapper class
            # This ensures sync API never deals with async directly → LSP preserved.
            raise RuntimeError(
                "authen_user() received async function; "
                "use AuthUserPassServerAsync for async usage."
            )

        return self.fn(username, password)

import asyncio
from greenlet import greenlet

async def greenlet_spawn(func, *args, **kwargs):
    loop = asyncio.get_running_loop()
    fut = loop.create_future()

    def run():
        try:
            res = func(*args, **kwargs)
            loop.call_soon_threadsafe(fut.set_result, res)
        except BaseException as e:
            loop.call_soon_threadsafe(fut.set_exception, e)

    g = greenlet(run)
    g.switch()
    return await fut

class AuthUserPassServerAsync:
    def __init__(self, sync_server: AuthUserPassServer):
        self.sync_server = sync_server

    async def authen_user(self, username, password):
        fn = self.sync_server.fn

        # Case A: async function provided
        if inspect.iscoroutinefunction(fn):
            return await fn(username, password)

        # Case B: sync function provided → offload to greenlet
        return await greenlet_spawn(fn, username, password)

    async def issue_token(self, *args, **kwargs):
        return await greenlet_spawn(self.sync_server.issue_token, *args, **kwargs)

    async def verify_token(self, token):
        return await greenlet_spawn(self.sync_server.verify_token, token)


# import inspect
# import asyncio
# from functools import partial

# class AuthUserPassServer(AuthServer):
#     def __init__(self, fn_check_userpass):
#         assert fn_check_userpass, "Must set fn_check_userpass"
#         self.fn = fn_check_userpass

#     def authen_user(self, username, password):
#         """Synchronous API — always safe to call."""
#         if inspect.iscoroutinefunction(self.fn):
#             # Run async fn inside thread-safe executor
#             loop = asyncio.get_event_loop()
#             return loop.run_until_complete(
#                 self.async_authen_user(username, password)
#             )
#         else:
#             return self.fn(username, password)

#     async def async_authen_user(self, username, password):
#         """Async API — never blocks."""
#         if inspect.iscoroutinefunction(self.fn):
#             return await self.fn(username, password)

#         # Offload sync function to a background thread
#         loop = asyncio.get_running_loop()
#         return await loop.run_in_executor(
#             None,
#             partial(self.fn, username, password)
# )


# ## Version `greenlet`
# import asyncio
# from greenlet import greenlet

# # Greenlet utility, similar to SQLAlchemy's `greenlet_spawn`
# async def greenlet_spawn(func, *args, **kwargs):
#     loop = asyncio.get_running_loop()
#     fut = loop.create_future()

#     def run_in_greenlet():
#         try:
#             result = func(*args, **kwargs)
#             loop.call_soon_threadsafe(fut.set_result, result)
#         except BaseException as e:
#             loop.call_soon_threadsafe(fut.set_exception, e)

#     g = greenlet(run_in_greenlet)
#     g.switch()
#     return await fut


# class AuthUserPassServer(AuthServer):
#     def __init__(self, fn_check_userpass):
#         assert fn_check_userpass, "Must set fn_check_userpass"
#         self.fn = fn_check_userpass

#     # -----------------------------------------
#     # 1) Synchronous API
#     # -----------------------------------------
#     def authen_user(self, username, password):
#         """Synchronous wrapper that never blocks the event loop."""
#         if inspect.iscoroutinefunction(self.fn):
#             # Sync API calls async API (LSP preserved)
#             return asyncio.run(
#                 self.async_authen_user(username, password)
#             )
#         else:
#             # Direct sync call
#             return self.fn(username, password)

#     # -----------------------------------------
#     # 2) Asynchronous API
#     # -----------------------------------------
#     async def async_authen_user(self, username, password):
#         """Async API that supports both sync and async functions."""

#         if inspect.iscoroutinefunction(self.fn):
#             # Case A: fn is async → run its await in a greenlet
#             return await greenlet_spawn(
#                 lambda: asyncio.run(self.fn(username, password))
#             )

#         else:
#             # Case B: fn is sync → run sync fn in greenlet
#             return await greenlet_spawn(
#                 self.fn, username, password
#             )



class UnAuthorizedException(Exception):
    def __init__(self, *args):
        super().__init__("UnAuthorized.")




pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    truncated = password.encode("utf-8")[:72].decode("utf-8", errors="ignore")
    return pwd_context.hash(truncated)

# Specified in FastAPI and our application
from models.crud import get_user_by_username
from db_core.db import get_db_connection

async def fn_check_user(username, password):
    # db = await get_db_connection()

    gen = get_db_connection()
    db = await gen.__anext__()  # get the yielded db

    try:

        user_in_db = await get_user_by_username(db, username)

        if not user_in_db or not verify_password(password, user_in_db.password):
            raise UnAuthorizedException()
        return {
            "username": "OK",
            "email": "Yes"
        }
    finally:
        await gen.aclose()

auth_server = AuthUserPassServer(fn_check_user)

```## util.py
```python
from passlib.context import CryptContext


pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    # return pwd_context.hash(password[:72])
    # print(f"[DEBUG] type(password)={type(password)}, value={repr(password)}")
    # if isinstance(password, str):
    #     password = password[:72]
    # else:
    #     raise TypeError(f"Expected str for password, got {type(password)}")
    # return pwd_context.hash(password)
    truncated = password.encode("utf-8")[:72].decode("utf-8", errors="ignore")
    return pwd_context.hash(truncated)
```## __init__.py
```python

```## [TOBE-REMOVE]auth_in_fastapi.py
```python
from datetime import datetime, timedelta
from jose import JWTError, jwt



# JWT settings: Should be keep in secret place
SECRET_KEY = "supersecretkey123"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30


from .util import verify_password


def create_access_token(data: dict, expires_delta: timedelta | None = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)



# Implement OAuth workflow couple with FastAPI
from fastapi.security import OAuth2PasswordBearer
from fastapi import Depends, HTTPException, status

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

from models.crud import get_user_by_username

async def authenticate_user(db, user_name:str, password: str):
    user_in_db = await get_user_by_username(db, user_name)

    if not user_in_db or not verify_password(password, user_in_db.password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    token = create_user_token(user_in_db)
    return {"access_token": token, "token_type": "bearer"}

    # return user_in_db


def create_user_token(user: dict):
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    return create_access_token(
        data={"user": user.name}, expires_delta=access_token_expires
    )

# async def get_current_user(token: str = Depends(oauth2_scheme)):
#     credentials_exception = HTTPException(
#         status_code=status.HTTP_401_UNAUTHORIZED,
#         detail="Could not validate credentials",
#         headers={"WWW-Authenticate": "Bearer"},
#     )

#     try:
#         payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
#         username = payload.get("user")
#         if username is None:
#             raise credentials_exception

#         user = await get_user_by_username(username)
#         if user is None:
#             raise credentials_exception

#         return user
#     except JWTError:
#         raise credentials_exception

from models.crud import create_post
async def create_post_with_auth(content, db, token):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("user")
        if username is None:
            raise credentials_exception

        post = await create_post(db, content)


        return post
    except JWTError:
        raise credentials_exception

```## env.py
```python
from logging.config import fileConfig

# from sqlalchemy import engine_from_config # No need
from sqlalchemy import pool # No need

from alembic import context

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

# Interpret the config file for Python logging.
# This line sets up loggers basically.
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata


# HoHai: I want to import models from sources. Then becareful with
# with the PYTHONPATH which setup at runtime to allow python can
# discover python package
from models import models
target_metadata = models.expose_base_meta()


from db_core.db import engine

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.


def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode.

    This configures the context with just a URL
    and not an Engine, though an Engine is acceptable
    here as well.  By skipping the Engine creation
    we don't even need a DBAPI to be available.

    Calls to context.execute() here emit the given string to the
    script output.

    """
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    # async with engine.begin() as conn:


    # with context.begin_transaction():
    #     context.run_migrations()


async def run_migrations_online() -> None:
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """

    # Option 1: Create engine from config
    # connectable = engine_from_config(
    #     config.get_section(config.config_ini_section, {}),
    #     prefix="sqlalchemy.",
    #     poolclass=pool.NullPool,
    # context.configure(
    #     connection=connection, target_metadata=target_metadata
    # )

    # with context.begin_transaction():
    #     context.run_migrations()
    # )

    # Option 2: reuse from db which we want to share

    print(target_metadata.tables)
    """Run migrations in 'online' (async) mode."""
    async with engine.connect() as conn:
        await conn.run_sync(do_run_migrations)


def do_run_migrations(connection):

    """Helper function to actually run migrations inside sync context."""
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,              # detect column type changes
        compare_server_default=True,    # detect default value changes
    )

    with context.begin_transaction():
        context.run_migrations()






# if context.is_offline_mode():
#     run_migrations_offline()
# else:
#     run_migrations_online()


import asyncio
asyncio.run(run_migrations_online())
```## 0002b4bcec43_init_database.py
```python
"""init database

Revision ID: 0002b4bcec43
Revises:
Create Date: 2025-11-10 19:34:30.997117

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = '0002b4bcec43'
down_revision: Union[str, Sequence[str], None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    """Upgrade schema."""
    # ### commands auto generated by Alembic - please adjust! ###
    op.create_table('posts',
    sa.Column('id', sa.Integer(), nullable=False),
    sa.Column('content', sa.String(length=100), nullable=True),
    sa.PrimaryKeyConstraint('id')
    )
    op.create_index(op.f('ix_posts_id'), 'posts', ['id'], unique=False)
    op.create_table('users',
    sa.Column('id', sa.Integer(), nullable=False),
    sa.Column('name', sa.String(length=50), nullable=True),
    sa.Column('email', sa.String(length=120), nullable=True),
    sa.Column('address', sa.String(length=250), nullable=True),
    sa.Column('password', sa.String(length=100), nullable=True),
    sa.PrimaryKeyConstraint('id')
    )
    op.create_index(op.f('ix_users_email'), 'users', ['email'], unique=True)
    op.create_index(op.f('ix_users_id'), 'users', ['id'], unique=False)
    # ### end Alembic commands ###


def downgrade() -> None:
    """Downgrade schema."""
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_index(op.f('ix_users_id'), table_name='users')
    op.drop_index(op.f('ix_users_email'), table_name='users')
    op.drop_table('users')
    op.drop_index(op.f('ix_posts_id'), table_name='posts')
    op.drop_table('posts')
    # ### end Alembic commands ###

```## models.py
```python
from sqlalchemy import Column, Integer, String
from db_core.db import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(50))
    email = Column(String(120), unique=True, index=True)
    address = Column(String(250)) # We want to add new columns
    password = Column(String(100))


# HoHai test
class Posts(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(String(100))


def expose_base_meta():
    return Base.metadata
```## schemas.py
```python
from pydantic import BaseModel, EmailStr

class UserBase(BaseModel):
    name: str
    password: str
    email: EmailStr

class UserCreate(UserBase):
    pass

class UserRead(UserBase):
    id: int

    class Config:
        orm_mode = True  # allows conversion from SQLAlchemy objects

```## crud.py
```python
from sqlalchemy.orm import Session
from models import models

from models import schemas

def get_user(db: Session, user_id: int):
    return db.query(models.User).filter(models.User.id == user_id).first()

# def get_users(db: Session, skip: int = 0, limit: int = 10):
#     return db.query(models.User).offset(skip).limit(limit).all()

from sqlalchemy.future import select




# Update in auth: we can expose to utitilty
# from auth.util import get_password_hash



async def get_users(db, skip: int = 0, limit: int = 100):
    result = await db.execute(select(models.User).offset(skip).limit(limit))
    return result.scalars().all()

# Actually db is not Session in the case AsyncSession (AsyncSession will wrap a Session instance)
async def create_user(db: Session, user: schemas.UserCreate):
    db_user = models.User(name=user.name, email=user.email, password=user.password)
    db.add(db_user)
    await db.commit() # Should await here
    await db.refresh(db_user)
    return db_user


# Update a use
# Bad behavior: no denote types in parameter
async def update_user(db: Session, username: str, user):

    # user_in_db = db.query(models.User).filter(models.User.id == user_id).first()

    # query_results = await db.execute(select(models.User).where(models.User.id == user_id))

    # user_in_db = query_results.scalar_one_or_none()
    user_in_db = await get_user_by_username(db, username)

    if user_in_db:
        # Update
        user_in_db.email = user.email
        user_in_db.name = user.name
        user_in_db.password = user.password


    await db.commit()
    return user_in_db


async def delete_user(db: Session, username: str):
    # Fetch the user
    # db_user = await db.get(models.User, user_id)
    db_user = await get_user_by_username(db, username)
    if db_user is None:
        return None  # or raise an exception

    # Delete the user
    await db.delete(db_user)
    await db.commit()  # must await

    return db_user  # returning the deleted object is optional


# add new method: getting user by user_name. Assume that username is unique in our app
async def get_user_by_username(db: Session, username):
    result = await db.execute(select(models.User).where(models.User.name == username))
    return result.scalar_one_or_none()


# Create/update post
async def create_post(db: Session, content: str):
    post = models.Posts(content=content)
    db.add(post)
    await db.commit()
    await db.refresh(post)
    return post

# async def get_posts(db:Session):
#     result = await db.execute(select(models.Posts))


```## db.py
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker


# Need to keep in secret place, don't hardcode like this
# Format: postgresql+psycopg2://user:password@host:port/database
DATABASE_URL = "postgresql+asyncpg://root:postgres@localhost:5432/example_pgdb"


engine = create_async_engine(DATABASE_URL, echo=True)





AsyncSessionLocal = sessionmaker(
    engine, expire_on_commit=False, class_=AsyncSession
)


async def get_db_connection():
    db = AsyncSessionLocal() # Create an instance for every calling time
    try:
        yield db
    finally:
        await db.close()



from sqlalchemy.ext.declarative import declarative_base


Base = declarative_base()



async def init_models():
    async with engine.begin() as conn:
        # Run the synchronous `create_all` within the async context
        await conn.run_sync(Base.metadata.create_all)

# import asyncio
# # Run the async function
# if __name__ == "__main__":
#     asyncio.run(init_models())


```## __init__.py
```python

```## __init__.py
```python

```## main.py
```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session


from models import schemas


from db_core.db import get_db_connection

import uvicorn

from models import crud

# Create tables
# Remove (refer to init_models)
# Base.metadata.create_all(bind=engine)

app = FastAPI()

@app.post("/users/", response_model=schemas.UserRead)
async def create_user(user: schemas.UserCreate, db: Session = Depends(get_db_connection)):
    db_user = await crud.create_user(db=db, user=user)


    return db_user

@app.get("/users/")
async def read_users(skip: int = 0, limit: int = 10, db: Session = Depends(get_db_connection)):
    users = await crud.get_users(db, skip=skip, limit=limit)

    return users

@app.get("/users/{user_id}", response_model=schemas.UserRead)
async def read_user(user_id: int, db: Session = Depends(get_db_connection)):
    db_user = await crud.get_user(db, user_id=user_id)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user

# Bad behavior
@app.post("/users/{username}")
async def update_user(username: str, user: schemas.UserCreate, db: Session = Depends(get_db_connection)):
    db_user = await crud.update_user(db, username=username, user=user)
    return db_user




# expose auth API
from auth.auth_in_fastapi import authenticate_user
from fastapi.security import OAuth2PasswordRequestForm
from fastapi import status

@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends(), db: Session = Depends(get_db_connection)):
    token_object = await authenticate_user(db, form_data.username, form_data.password)
    return token_object


from auth.auth_in_fastapi import oauth2_scheme, create_post_with_auth
@app.post("/post")
async def create_post_with_protected(content: str, db: Session = Depends(get_db_connection), token: str = Depends(oauth2_scheme)):
    result = await create_post_with_auth(content=content, db=db, token=token)
    return result



if __name__ == "__main__":
    # TODO: Should be run here to make sure that all tables (which mapped with models) will be scan in `metadata`
    # asyncio.run(init_models())



    # uvicorn.run(app, host="127.0.0.1", port=8000, reload=True)
    uvicorn.run("main:app", host="127.0.0.1", port=8000, reload=True)
    # uvicorn.run("main:app", host="127.0.0.1", port=8000)
```## auth.py
```python
# Build AuthServer
from datetime import datetime, timedelta
from jose import JWTError, jwt
import inspect
from passlib.context import CryptContext

class AuthServer:
    def __init__(self):
        # You can have more flexibilites by putting these info in contructor
        self._access_token_expire_mins = 30
        self._secret_key = "supersecretkey123"
        self._algorithm = "HS256"

    def authen_user(self, *args, **kwargs) -> dict:
        raise NotImplementedError()

    def issue_token(self, *args, **kwargs):
        user_dict = self.authen_user(*args, **kwargs)
        if user_dict is None:
            raise Exception("User not authenticated")

        to_encode = user_dict.copy()
        expire = datetime.now() + timedelta(minutes=self._access_token_expire_mins)
        to_encode.update({"exp": expire})
        return jwt.encode(to_encode, self._secret_key, algorithm=self._algorithm)

    def verify_token(self, token: str) -> dict:
        return jwt.decode(token, self._secret_key, algorithms=[self._algorithm])


class AuthUserPassServer(AuthServer):
    def __init__(self, fn_check_userpass):
        super().__init__()
        assert fn_check_userpass is not None, "Must set `fn_check_userpass`"
        self.fn = fn_check_userpass

    # Overriding sync with async breaks Liskov Substitution
    def authen_user(self, username, password) -> dict:
        # Support async functions
        if inspect.iscoroutinefunction(self.fn):
            import asyncio
            import nest_asyncio

            # Support jupyter
            nest_asyncio.apply()

            loop = asyncio.get_event_loop()
            task = loop.create_task(self.fn(username, password))
            loop.run_until_complete(task)
            return task.result()
        else:
            return self.fn(username, password)

class UnAuthorizedException(Exception):
    def __init__(self, *args):
        super().__init__("UnAuthorized.")




pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    truncated = password.encode("utf-8")[:72].decode("utf-8", errors="ignore")
    return pwd_context.hash(truncated)

# Specified in FastAPI and our application
from models.crud import get_user_by_username
from db_core.db import get_db_connection

async def fn_check_user(user_name, password):
    db = await get_db_connection()
    user_in_db = await get_user_by_username(db, user_name)
    if not user_name or verify_password(password, user_in_db.password):
        raise UnAuthorizedException()
    return {
        "user": user_in_db
    }


```## util.py
```python
from passlib.context import CryptContext


pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    # return pwd_context.hash(password[:72])
    # print(f"[DEBUG] type(password)={type(password)}, value={repr(password)}")
    # if isinstance(password, str):
    #     password = password[:72]
    # else:
    #     raise TypeError(f"Expected str for password, got {type(password)}")
    # return pwd_context.hash(password)
    truncated = password.encode("utf-8")[:72].decode("utf-8", errors="ignore")
    return pwd_context.hash(truncated)
```## __init__.py
```python

```## auth_in_fastapi.py
```python
from datetime import datetime, timedelta
from jose import JWTError, jwt



# JWT settings: Should be keep in secret place
SECRET_KEY = "supersecretkey123"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30


from .util import verify_password


def create_access_token(data: dict, expires_delta: timedelta | None = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)



# Implement OAuth workflow couple with FastAPI
from fastapi.security import OAuth2PasswordBearer
from fastapi import Depends, HTTPException, status

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

from models.crud import get_user_by_username

async def authenticate_user(db, user_name:str, password: str):
    user_in_db = await get_user_by_username(db, user_name)

    if not user_in_db or not verify_password(password, user_in_db.password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    token = create_user_token(user_in_db)
    return {"access_token": token, "token_type": "bearer"}

    # return user_in_db


def create_user_token(user: dict):
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    return create_access_token(
        data={"user": user.name}, expires_delta=access_token_expires
    )

# async def get_current_user(token: str = Depends(oauth2_scheme)):
#     credentials_exception = HTTPException(
#         status_code=status.HTTP_401_UNAUTHORIZED,
#         detail="Could not validate credentials",
#         headers={"WWW-Authenticate": "Bearer"},
#     )

#     try:
#         payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
#         username = payload.get("user")
#         if username is None:
#             raise credentials_exception

#         user = await get_user_by_username(username)
#         if user is None:
#             raise credentials_exception

#         return user
#     except JWTError:
#         raise credentials_exception

from models.crud import create_post
async def create_post_with_auth(content, db, token):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("user")
        if username is None:
            raise credentials_exception

        post = await create_post(db, content)


        return post
    except JWTError:
        raise credentials_exception

```## env.py
```python
from logging.config import fileConfig

# from sqlalchemy import engine_from_config # No need
from sqlalchemy import pool # No need

from alembic import context

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

# Interpret the config file for Python logging.
# This line sets up loggers basically.
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata


# HoHai: I want to import models from sources. Then becareful with
# with the PYTHONPATH which setup at runtime to allow python can
# discover python package
from models import models
target_metadata = models.expose_base_meta()


from db_core.db import engine

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.


def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode.

    This configures the context with just a URL
    and not an Engine, though an Engine is acceptable
    here as well.  By skipping the Engine creation
    we don't even need a DBAPI to be available.

    Calls to context.execute() here emit the given string to the
    script output.

    """
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    # async with engine.begin() as conn:


    # with context.begin_transaction():
    #     context.run_migrations()


async def run_migrations_online() -> None:
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """

    # Option 1: Create engine from config
    # connectable = engine_from_config(
    #     config.get_section(config.config_ini_section, {}),
    #     prefix="sqlalchemy.",
    #     poolclass=pool.NullPool,
    # context.configure(
    #     connection=connection, target_metadata=target_metadata
    # )

    # with context.begin_transaction():
    #     context.run_migrations()
    # )

    # Option 2: reuse from db which we want to share

    print(target_metadata.tables)
    """Run migrations in 'online' (async) mode."""
    async with engine.connect() as conn:
        await conn.run_sync(do_run_migrations)


def do_run_migrations(connection):

    """Helper function to actually run migrations inside sync context."""
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,              # detect column type changes
        compare_server_default=True,    # detect default value changes
    )

    with context.begin_transaction():
        context.run_migrations()






# if context.is_offline_mode():
#     run_migrations_offline()
# else:
#     run_migrations_online()


import asyncio
asyncio.run(run_migrations_online())
```## 0002b4bcec43_init_database.py
```python
"""init database

Revision ID: 0002b4bcec43
Revises:
Create Date: 2025-11-10 19:34:30.997117

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = '0002b4bcec43'
down_revision: Union[str, Sequence[str], None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    """Upgrade schema."""
    # ### commands auto generated by Alembic - please adjust! ###
    op.create_table('posts',
    sa.Column('id', sa.Integer(), nullable=False),
    sa.Column('content', sa.String(length=100), nullable=True),
    sa.PrimaryKeyConstraint('id')
    )
    op.create_index(op.f('ix_posts_id'), 'posts', ['id'], unique=False)
    op.create_table('users',
    sa.Column('id', sa.Integer(), nullable=False),
    sa.Column('name', sa.String(length=50), nullable=True),
    sa.Column('email', sa.String(length=120), nullable=True),
    sa.Column('address', sa.String(length=250), nullable=True),
    sa.Column('password', sa.String(length=100), nullable=True),
    sa.PrimaryKeyConstraint('id')
    )
    op.create_index(op.f('ix_users_email'), 'users', ['email'], unique=True)
    op.create_index(op.f('ix_users_id'), 'users', ['id'], unique=False)
    # ### end Alembic commands ###


def downgrade() -> None:
    """Downgrade schema."""
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_index(op.f('ix_users_id'), table_name='users')
    op.drop_index(op.f('ix_users_email'), table_name='users')
    op.drop_table('users')
    op.drop_index(op.f('ix_posts_id'), table_name='posts')
    op.drop_table('posts')
    # ### end Alembic commands ###

```## models.py
```python
from sqlalchemy import Column, Integer, String
from db_core.db import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(50))
    email = Column(String(120), unique=True, index=True)
    address = Column(String(250)) # We want to add new columns
    password = Column(String(100))


# HoHai test
class Posts(Base):
    __tablename__ = "posts"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(String(100))


def expose_base_meta():
    return Base.metadata
```## schemas.py
```python
from pydantic import BaseModel, EmailStr

class UserBase(BaseModel):
    name: str
    password: str
    email: EmailStr

class UserCreate(UserBase):
    pass

class UserRead(UserBase):
    id: int

    class Config:
        orm_mode = True  # allows conversion from SQLAlchemy objects

```## crud.py
```python
from sqlalchemy.orm import Session
from models import models

from models import schemas

def get_user(db: Session, user_id: int):
    return db.query(models.User).filter(models.User.id == user_id).first()

# def get_users(db: Session, skip: int = 0, limit: int = 10):
#     return db.query(models.User).offset(skip).limit(limit).all()

from sqlalchemy.future import select




# Update in auth: we can expose to utitilty
from auth.util import get_password_hash



async def get_users(db, skip: int = 0, limit: int = 100):
    result = await db.execute(select(models.User).offset(skip).limit(limit))
    return result.scalars().all()

# Actually db is not Session in the case AsyncSession (AsyncSession will wrap a Session instance)
async def create_user(db: Session, user: schemas.UserCreate):
    db_user = models.User(name=user.name, email=user.email, password=get_password_hash(user.password))
    db.add(db_user)
    await db.commit() # Should await here
    await db.refresh(db_user)
    return db_user


# Update a use
# Bad behavior: no denote types in parameter
async def update_user(db: Session, username: str, user):

    # user_in_db = db.query(models.User).filter(models.User.id == user_id).first()

    # query_results = await db.execute(select(models.User).where(models.User.id == user_id))

    # user_in_db = query_results.scalar_one_or_none()
    user_in_db = await get_user_by_username(db, username)

    if user_in_db:
        # Update
        user_in_db.email = user.email
        user_in_db.name = user.name


        user_in_db.password = get_password_hash(user.password)


    await db.commit()
    return user_in_db

# add new method: getting user by user_name. Assume that username is unique in our app
async def get_user_by_username(db: Session, username):
    result = await db.execute(select(models.User).where(models.User.name == username))
    return result.scalar_one_or_none()


# Create/update post
async def create_post(db: Session, content: str):
    post = models.Posts(content=content)
    db.add(post)
    await db.commit()
    await db.refresh(post)
    return post

# async def get_posts(db:Session):
#     result = await db.execute(select(models.Posts))


```## db.py
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker


# Need to keep in secret place, don't hardcode like this
# Format: postgresql+psycopg2://user:password@host:port/database
DATABASE_URL = "postgresql+asyncpg://root:postgres@localhost:5432/example_pgdb"


engine = create_async_engine(DATABASE_URL, echo=True)





AsyncSessionLocal = sessionmaker(
    engine, expire_on_commit=False, class_=AsyncSession
)


async def get_db_connection():
    db = AsyncSessionLocal() # Create an instance for every calling time
    try:
        yield db
    finally:
        await db.close()



from sqlalchemy.ext.declarative import declarative_base


Base = declarative_base()



async def init_models():
    async with engine.begin() as conn:
        # Run the synchronous `create_all` within the async context
        await conn.run_sync(Base.metadata.create_all)

# import asyncio
# # Run the async function
# if __name__ == "__main__":
#     asyncio.run(init_models())


```## __init__.py
```python

```## wrap_async_inside_normal_function.py
```python
import asyncio
import inspect

import nest_asyncio
nest_asyncio.apply()  # only needed in Jupyter

async def fn_check_user_pass_async(u, p):
    await asyncio.sleep(1.0)
    return {"user": u, "password": p}

# asyncio.get_event_loop().run_until_complete(coro)


def wrap_fn():
    if inspect.iscoroutinefunction(fn_check_user_pass_async):
        loop = asyncio.get_event_loop()
        task = loop.create_task(fn_check_user_pass_async("HaiHT", "123"))
        loop.run_until_complete(task)

        print(task.result())

    else:
        print(fn_check_user_pass_async("HaiHT", "123"))

wrap_fn()

```## something.py
```python
import asyncio
import threading

def worker():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    print("Worker thread loop:", id(asyncio.get_running_loop()))


threading.Thread(target=worker).start()

main_loop = asyncio.new_event_loop()
asyncio.set_event_loop(main_loop)
print("Main thread loop:", id(asyncio.get_running_loop()))
```
