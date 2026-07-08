# em-nodejs — Rules Reference

## 1. Architecture & Structure
| Rule | Description | Severity |
|---|---|---|
| ND-01 | Controllers are thin — delegate all logic to services; no business rules in controllers | 🔴 |
| ND-02 | Services depend on repository interfaces — never import ORM/Prisma directly in services | 🟠 |
| ND-03 | Constructor injection only — no `new ConcreteClass()` inside services or controllers | 🟠 |
| ND-04 | One NestJS module per domain feature — no cross-module repository imports | 🟠 |
| ND-05 | No circular module dependencies | 🔴 |

## 2. Validation & Input
| Rule | Description | Severity |
|---|---|---|
| ND-06 | Global `ValidationPipe` with `whitelist: true, forbidNonWhitelisted: true, transform: true` | 🔴 |
| ND-07 | DTOs with `class-validator` decorators for every request body | 🟠 |
| ND-08 | No raw `req.body` or `req.query` used directly without typed DTO | 🟠 |

## 3. Auth & Guards
| Rule | Description | Severity |
|---|---|---|
| ND-09 | `JwtAuthGuard` applied globally — `@Public()` for explicit opt-out | 🔴 |
| ND-10 | `RolesGuard` reads role from JWT payload only — never from request body | 🔴 |
| ND-11 | `ThrottlerModule` rate limiting on auth and public endpoints | 🔴 |

## 4. Error Handling
| Rule | Description | Severity |
|---|---|---|
| ND-12 | Global `ExceptionFilter` returns consistent error envelope — no stack traces to client | 🟠 |
| ND-13 | Domain-specific error classes thrown from services — HTTP errors only in filters | 🟡 |
| ND-14 | No swallowed exceptions: `catch (e) {}` | 🔴 |
| ND-15 | Every error response includes `traceId` for support traceability | 🟡 |

## 5. Logging
| Rule | Description | Severity |
|---|---|---|
| ND-16 | No `console.log` — structured JSON logger (Pino/Winston) only | 🟡 |
| ND-17 | No PII (email, phone, password, token) in any log statement | 🔴 |
| ND-18 | `traceId` propagated from request headers through all log lines | 🟠 |
| ND-19 | Log level configurable per environment — `warn` in production | 🟠 |

## 6. Performance & Safety
| Rule | Description | Severity |
|---|---|---|
| ND-20 | No blocking sync I/O (`readFileSync`, `execSync`) in request handlers | 🟠 |
| ND-21 | DB and Redis connections initialised at module startup — never per request | 🔴 |
| ND-22 | `SIGTERM`/`SIGINT` handler for graceful shutdown | 🟠 |
| ND-23 | All external calls have timeout + retry with exponential backoff | 🟠 |
| ND-24 | No N+1 queries — batch or use DataLoader | 🔴 |
| ND-25 | All list endpoints paginated — cursor-based for large datasets | 🔴 |
| ND-26 | `Promise.all([...])` for independent async operations | 🟠 |

## 7. TypeScript
| Rule | Description | Severity |
|---|---|---|
| ND-27 | `strict: true` in every `tsconfig.json` | 🔴 |
| ND-28 | No `any` type — `unknown` with type guard or proper interface | 🟠 |
| ND-29 | Explicit return types on all public service methods | 🟡 |

## 8. Testing
| Rule | Description | Severity |
|---|---|---|
| ND-30 | Unit tests for all service methods — happy path + errors + edge cases | 🔴 |
| ND-31 | All dependencies mocked — tests never hit real DB or external APIs | 🟠 |
| ND-32 | Coverage ≥ 80% on service layer; ≥ 90% on domain/business logic | 🟠 |
| ND-33 | SonarQube integrated and quality gate blocks CI merge | 🟠 |

## 9. Instant Escalation — 🔴
| # | Violation |
|---|---|
| ESC-01 | No `ValidationPipe` — raw unvalidated input reaches handlers |
| ESC-02 | PII in any log statement |
| ESC-03 | Blocking sync I/O in async request handler |
| ESC-04 | New DB connection created per request |
| ESC-05 | `eval(userInput)` or `exec` with user-controlled input |
| ESC-06 | No auth guard on state-changing route |
| ESC-07 | Swallowed exception: `catch (e) {}` |
