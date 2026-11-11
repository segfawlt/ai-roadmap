# JWT Implementation Reference

Complete reference implementation for JWT-based authentication in FastAPI.

## auth/jwt_handler.py

``` python
from datetime import datetime, timedelta
from jose import JWTError, jwt
from pydantic import BaseModel

# Configuration — in production, load from environment variables
SECRET_KEY = "your-secret-key-at-least-32-characters-long"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 15
REFRESH_TOKEN_EXPIRE_DAYS = 7


class TokenData(BaseModel):
    sub: str | None = None


class TokenPair(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))
    to_encode.update({"exp": expire, "type": "access"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def create_refresh_token(data: dict) -> str:
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    to_encode.update({"exp": expire, "type": "refresh"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def create_token_pair(user_id: int) -> TokenPair:
    """Create both access and refresh tokens for a user."""
    data = {"sub": str(user_id)}
    return TokenPair(
        access_token=create_access_token(data),
        refresh_token=create_refresh_token(data),
    )


def verify_token(token: str, expected_type: str = "access") -> TokenData:
    """Verify and decode a JWT token.

    Raises JWTError if token is invalid, expired, or wrong type.
    """
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    token_type = payload.get("type")
    if token_type != expected_type:
        raise JWTError(f"Expected {expected_type} token, got {token_type}")
    sub: str = payload.get("sub")
    if sub is None:
        raise JWTError("Token missing 'sub' claim")
    return TokenData(sub=sub)
```

## auth/password.py

``` python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)
```

## dependencies.py

``` python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError
from sqlalchemy.ext.asyncio import AsyncSession

from auth.jwt_handler import verify_token
from db import get_db
from models import User
from sqlalchemy import select

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")


async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        token_data = verify_token(token, expected_type="access")
    except JWTError:
        raise credentials_exception

    result = await db.execute(select(User).where(User.id == int(token_data.sub)))
    user = result.scalars().first()
    if user is None:
        raise credentials_exception
    return user
```

## routers/auth.py

``` python
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from auth.jwt_handler import create_token_pair, verify_token, TokenPair
from auth.password import hash_password, verify_password
from db import get_db
from models import User
from schemas import UserCreate, UserResponse

router = APIRouter(prefix="/auth", tags=["auth"])


@router.post("/register", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def register(user_data: UserCreate, db: AsyncSession = Depends(get_db)):
    # Check if email already exists
    result = await db.execute(select(User).where(User.email == user_data.email))
    if result.scalars().first():
        raise HTTPException(status_code=409, detail="Email already registered")

    user = User(
        name=user_data.name,
        email=user_data.email,
        hashed_password=hash_password(user_data.password),
    )
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return user


@router.post("/login", response_model=TokenPair)
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(select(User).where(User.email == form_data.username))
    user = result.scalars().first()
    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
        )
    return create_token_pair(user.id)


@router.post("/refresh", response_model=TokenPair)
async def refresh_token(refresh_token: str, db: AsyncSession = Depends(get_db)):
    try:
        token_data = verify_token(refresh_token, expected_type="refresh")
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid refresh token")

    result = await db.execute(select(User).where(User.id == int(token_data.sub)))
    user = result.scalars().first()
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    return create_token_pair(user.id)
```

## Key Design Decisions

| Decision | Rationale |
|----|----|
| Separate access + refresh tokens | Short-lived access (15 min) limits damage if stolen; long-lived refresh (7 days) avoids frequent re-login |
| Token type in payload (`"type":`` ``"access"`) | Prevents using a refresh token as an access token |
| `sub` claim as string user ID | Standard JWT claim; only non-sensitive identifier stored |
| bcrypt for password hashing | Industry standard with configurable work factor |
| `OAuth2PasswordBearer` dependency | Integrates with FastAPI’s auto-generated Swagger auth UI |
