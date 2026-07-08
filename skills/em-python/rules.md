# em-python — Rules Reference

## 1. Code Style & Structure
| Rule | Description | Severity |
|---|---|---|
| PY-01 | All public functions have type hints — `mypy --strict` clean | 🟡 |
| PY-02 | `snake_case` for variables/functions; `PascalCase` for classes; `UPPER_SNAKE_CASE` for constants | 🟡 |
| PY-03 | No bare `except:` — always catch specific exception types | 🟠 |
| PY-04 | No mutable default arguments — `def fn(items=None): items = items or []` | 🟠 |
| PY-05 | `with` statement for all file, DB, and connection handling | 🟠 |
| PY-06 | No wildcard imports `from module import *` | 🟡 |
| PY-07 | Functions ≤ 30 lines; classes ≤ 200 lines | 🟠 |
| PY-08 | f-strings for string formatting — not `%` or `.format()` | 🟡 |

## 2. FastAPI / Django
| Rule | Description | Severity |
|---|---|---|
| PY-09 | Pydantic model on every FastAPI endpoint — no `dict` or `Any` body | 🟠 |
| PY-10 | `response_model` set on every FastAPI endpoint — no raw entity return | 🟠 |
| PY-11 | `Depends(get_db)` for DB session injection — never global DB object | 🟠 |
| PY-12 | Async DB driver for `async def` endpoints — `asyncpg`, `motor`, `aioredis` | 🟠 |
| PY-13 | Django: `select_related`/`prefetch_related` to prevent N+1 | 🔴 |
| PY-14 | `DEBUG = False` in production Django settings | 🔴 |
| PY-15 | Rate limiting on auth endpoints — `slowapi` (FastAPI) or `django-ratelimit` | 🔴 |

## 3. Security
| Rule | Description | Severity |
|---|---|---|
| PY-16 | Parameterised queries only — no f-string or `%` interpolation in SQL | 🔴 |
| PY-17 | No `eval()`, `exec()`, or `pickle.loads()` with user input | 🔴 |
| PY-18 | No `subprocess.call(cmd, shell=True)` — use list form, `shell=False` | 🔴 |
| PY-19 | Secrets from environment variables — `python-decouple` or `pydantic-settings` | 🔴 |
| PY-20 | Passwords hashed with `bcrypt` or `argon2-cffi` — never MD5 or SHA1 | 🔴 |

## 4. Logging
| Rule | Description | Severity |
|---|---|---|
| PY-21 | No `print()` in production — structured logger (`logging` or `structlog`) | 🟡 |
| PY-22 | No PII (email, phone, password) in any log statement | 🔴 |
| PY-23 | Log level configurable per environment | 🟠 |
| PY-24 | Exceptions logged with `logger.exception(msg)` — includes stack trace automatically | 🟠 |

## 5. Performance
| Rule | Description | Severity |
|---|---|---|
| PY-25 | No N+1 queries — batch or use `select_related`/`prefetch_related` | 🔴 |
| PY-26 | All list endpoints paginated — never unbounded querysets | 🔴 |
| PY-27 | Connection pooling configured — `SQLAlchemy` pool or `databases` library | 🔴 |
| PY-28 | Heavy tasks in Celery queue — never block a request | 🟠 |
| PY-29 | `@lru_cache` or Redis TTL cache on expensive repeated computations | 🟠 |

## 6. Testing
| Rule | Description | Severity |
|---|---|---|
| PY-30 | `pytest` with `pytest-asyncio` for async tests | 🟠 |
| PY-31 | All external dependencies mocked — tests never hit real APIs or DBs | 🟠 |
| PY-32 | `factory_boy` for test fixtures — no hardcoded test objects | 🟡 |
| PY-33 | Coverage ≥ 80% on service layer; ≥ 90% on business logic | 🟠 |

## 7. Instant Escalation — 🔴
| # | Violation |
|---|---|
| ESC-01 | SQL query built with f-string or `%` interpolation |
| ESC-02 | `pickle.loads(user_bytes)` — arbitrary code execution |
| ESC-03 | `subprocess.call(cmd, shell=True)` with dynamic input |
| ESC-04 | PII in any log statement |
| ESC-05 | No Pydantic validation on FastAPI endpoint |
| ESC-06 | Bare `except:` swallowing all exceptions silently |
| ESC-07 | `os.system(f"cmd {user_input}")` — command injection |
