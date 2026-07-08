# Python — Gap Detection

## Code Quality Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Bare `except` | `except:` or `except Exception:` catching everything silently | 🟠 |
| Mutable default argument | `def fn(items=[]):` | 🟠 |
| Missing type hints | Public function without parameter/return type annotations | 🟡 |
| f-string not used | `"Hello " + name` or `"Hello %s" % name` instead of f-string | 🟡 |
| `open()` without `with` | `f = open("file")` not inside `with` block — resource leak | 🟠 |
| Wildcard import | `from module import *` | 🟡 |
| `print()` in production code | Debug prints left in source | 🟡 |
| No `__all__` on public module | Public API undeclared | 🟡 |

## FastAPI / Django Security Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No Pydantic model on endpoint | `@app.post` handler using `dict` or `Any` instead of Pydantic model | 🟠 |
| `allow_origins=["*"]` with credentials | CORS wildcard with `allow_credentials=True` | 🔴 |
| Raw SQL with f-string | `cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")` | 🔴 |
| No rate limiting on auth routes | `/login`, `/register` with no throttle middleware | 🔴 |
| Password stored as plain text or MD5 | `hashlib.md5(password)`, no bcrypt/argon2 | 🔴 |
| `DEBUG=True` in production | Django `DEBUG = True` in production settings | 🔴 |
| Secret key hardcoded | `SECRET_KEY = "hardcoded-value"` in settings | 🔴 |

## Performance Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Sync DB in async FastAPI endpoint | `db.query()` (sync SQLAlchemy) inside `async def` handler | 🟠 |
| N+1 in Django ORM | `for order in orders: order.user.name` — lazy load per iteration | 🔴 |
| No `select_related`/`prefetch_related` | Django queryset accessing related models in a loop | 🔴 |
| Missing `@lru_cache` / Redis on expensive fn | Pure function called repeatedly with same args and no caching | 🟡 |

## Generation Checklist
- [ ] All functions have type hints; `mypy --strict` clean
- [ ] All `except` clauses catch specific exception types
- [ ] `with` statement for all file/DB/connection handling
- [ ] Pydantic models for all FastAPI request/response schemas
- [ ] Async DB driver (`asyncpg`, `motor`) for async endpoints
- [ ] Secrets from environment variables; `python-decouple` or `pydantic-settings`
