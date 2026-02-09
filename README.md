# FastAPI Auth Kit

**FastAPI authentication library with JWT access/refresh tokens, sync & async SQLAlchemy support, built-in auth routes (register/login/refresh/me), secure password hashing, and cookie/header token support.**

Complete authentication toolkit for FastAPI with JWT, designed for both sync and async SQLAlchemy backends. Provides batteries-included auth services, FastAPI routers, token utilities, and a clean protocol-driven repository interface.

## Highlights

- JWT access + refresh tokens with rotation
- Sync and async SQLAlchemy adapters
- Drop-in FastAPI routers for register, login, refresh, logout, and `/me`
- Cookie and Authorization header support
- Strong typing and protocol-based extensibility
- Works with your own User model

## Package Layout

```
packages/authkit/src/authkit/
  fastapi/      # FastAPI router builders + schemas
  ext/          # SQLAlchemy protocol adapters (sync/async)
  authenticator.py
  service.py
  tokens.py
  hashing.py
  settings.py
  extractors.py
```

## Installation

### From this repo

```bash
uv sync
```

### In your project

```bash
pip install fastapi-auth-kit
```

> The PyPI package name is `fastapi-auth-kit`, the import name is `authkit`.

### Typing support

Type information (`.pyi` + `py.typed`) is bundled with the package for IDEs and
static type checkers.

## Quickstart (Async)

```python
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

from authkit import AuthSettings
from authkit.fastapi.routers import build_auth_router_async

from your_app.models import Base, User

DATABASE_URL = "sqlite+aiosqlite:///./app.db"
engine = create_async_engine(DATABASE_URL, echo=False)
SessionLocal = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_session():
    async with SessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

settings = AuthSettings(secret_key="change-me", cookie_secure=False)

app = FastAPI()
auth_router = build_auth_router_async(
    settings=settings,
    get_session=get_session,
    user_model=User,
)
app.include_router(auth_router, prefix="/auth", tags=["auth"])
```

## Quickstart (Sync)

```python
from fastapi import FastAPI
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from authkit import AuthSettings
from authkit.fastapi.routers import build_auth_router_sync

from your_app.models import Base, User

DATABASE_URL = "sqlite:///./app.db"
engine = create_engine(DATABASE_URL, echo=False)
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)

def get_session():
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

settings = AuthSettings(secret_key="change-me", cookie_secure=False)

app = FastAPI()
auth_router = build_auth_router_sync(
    settings=settings,
    get_session=get_session,
    user_models=User,
)
app.include_router(auth_router, prefix="/auth", tags=["auth"])
```

## Core Concepts

### AuthSettings

All auth behavior is configured via `AuthSettings`:

```python
AuthSettings(
    secret_key="your-secret",
    algorithm="HS256",
    access_minutes=15,
    refresh_days=7,
    accept_header=True,
    accept_cookie=True,
    set_cookie_on_login=True,
    cookie_secure=True,
    cookie_samesite="lax",
)
```

### AuthService / AsyncAuthService

Business logic for creating users, authenticating, and issuing tokens. Uses a
repository protocol so you can plug in your own persistence layer.

### SQLAlchemy Adapters

- `SQLAlchemySyncUserProtocol`
- `SQLAlchemyAsyncUserProtocol`

These adapters expect a SQLAlchemy model with the following fields:

```python
class User(Base):
    id: int
    email: str
    username: str
    password_hash: str
    is_active: bool
    is_staff: bool
    is_superuser: bool
```

## API Endpoints

When you include the auth router with prefix `/auth`, the following endpoints
are available:

| Method | Path           | Description                 |
|--------|----------------|-----------------------------|
| POST   | /auth/register | Create a new user           |
| POST   | /auth/login    | Authenticate + issue tokens |
| POST   | /auth/refresh  | Refresh access token        |
| POST   | /auth/logout   | Clear auth cookies          |
| GET    | /auth/me       | Get current user            |

### Request/Response Schemas

**Register**
```json
{ "email": "user@example.com", "username": "user", "password": "secret" }
```

**Login**
```json
{ "username_or_email": "user", "password": "secret" }
```

**Token Response**
```json
{ "access_token": "jwt", "refresh_token": "jwt" }
```

## Cookies and Headers

`AuthSettings` controls where tokens are accepted and how they are stored:

- `accept_header`: allow `Authorization: Bearer <token>`
- `accept_cookie`: allow cookies
- `set_cookie_on_login`: set cookies on successful login

Cookie names and TTL are customizable with:
`cookie_name_access`, `cookie_name_refresh`, `cookie_max_age_access`,
`cookie_max_age_refresh`.

## Security Considerations

- Use a strong `secret_key` and rotate regularly.
- Set `cookie_secure=True` in production (HTTPS only).
- Consider `cookie_samesite="strict"` for web apps with tight CSRF control.
- Short access token TTLs with longer refresh TTLs are recommended.

## Production Checklist

- [x] `secret_key` stored in a secure secret manager
- [x] HTTPS enforced
- [x] `cookie_secure=True`, `cookie_samesite` set per app policy
- [x] Rotate JWT secret or use KMS-backed signing
- [x] Enable logging around login and refresh flows
- [x] Implement account lockout or rate limiting at the API gateway
- [x] Configure backups for the user datastore

## Compatibility

- Python: 3.10+
- FastAPI: 0.110+
- SQLAlchemy: 2.x

## Observability

This library raises standard `HTTPException` errors. For production:

- Add structured logging around auth endpoints
- Add tracing/metrics at the FastAPI middleware layer

## Versioning

Follows semantic versioning: `MAJOR.MINOR.PATCH`.

## Testing

Run the full test suite:

```bash
uv run pytest tests/ -v
```

All tests are under `tests/` and cover tokens, hashing, services, and router
integration (sync + async).

## Examples

Working example apps are provided:

- `examples/async_app.py`
- `examples/sync_app.py`

Run:

```bash
uv run python -m examples.async_app
uv run python -m examples.sync_app
```

## Extending the Repo Layer

Implement the repo protocol if you use a different persistence layer:

```python
class MyRepo:
    def get_by_id(self, user_id: int): ...
    def get_by_email_or_username(self, value: str): ...
    def create_user(self, *, email: str, username: str, password: str,
                    is_staff: bool, is_active: bool, is_superuser: bool): ...
```

Async version must expose the same methods as `async def`.

## Typing Notes

Type information is bundled with the package (`.pyi` + `py.typed`). If your IDE
or type checker does not pick it up, ensure:

- Your tooling supports PEP 561

## Troubleshooting

**Bcrypt errors on Windows**

If you see bcrypt backend errors during hashing, ensure you have `bcrypt<5`
installed. This repo pins it accordingly in `pyproject.toml`.

**SQLite in-memory tests**

In-memory SQLite needs a `StaticPool` so all sessions share the same DB
connection. The tests already handle this.

## License

MIT (see `LICENSE`).
