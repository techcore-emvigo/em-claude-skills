# em-code-quality — Rules Reference

Non-negotiable rules enforced during every code review and generation task.
Prefix: **CQ** = Code Quality | **NM** = Naming | **EH** = Error Handling |
**LG** = Logging | **TS** = Type Safety | **PF** = Performance | **TT** = Testing

---

## 1. Code Structure & Responsibility

| Rule | Description | Severity |
|---|---|---|
| CQ-01 | One responsibility per function — max **30 lines** per function body | 🔴 |
| CQ-02 | One responsibility per class — max **200 lines** per file | 🟠 |
| CQ-03 | Business logic in service/domain layer only — never in controllers, routes, or DB layer | 🔴 |
| CQ-04 | Dependencies injected via constructor — never `new ConcreteClass()` inside a service (DIP) | 🟠 |
| CQ-05 | Interfaces at every layer boundary — domain never imports infrastructure or framework types | 🔴 |
| CQ-06 | No circular dependencies between modules or layers | 🔴 |
| CQ-07 | Max 3 nesting levels — use early returns to flatten deep if/for/try blocks | 🟡 |
| CQ-08 | No copy-paste duplication — extract shared logic into a reusable function or class (DRY) | 🟠 |
| CQ-09 | Remove all dead code — commented-out blocks, unused variables, unused imports | 🟡 |
| CQ-10 | Max 4 function parameters — use a DTO/options object beyond 4 | 🟡 |
| CQ-11 | No boolean trap — `createUser(data, true, false)` must use named options object | 🟡 |
| CQ-12 | SOLID — SRP: class has one reason to change; OCP: extend via abstraction not modification | 🟠 |

---

## 2. Naming Conventions

| Rule | Description | Severity |
|---|---|---|
| NM-01 | Names must be intention-revealing — purpose clear without reading the implementation | 🟠 |
| NM-02 | JS/TS: `camelCase` variables/functions, `PascalCase` classes, `UPPER_SNAKE_CASE` constants | 🟡 |
| NM-03 | Python: `snake_case` variables/functions, `PascalCase` classes, `UPPER_SNAKE_CASE` constants | 🟡 |
| NM-04 | No abbreviations unless universally known (`id`, `url`, `api`, `http`) | 🟡 |
| NM-05 | No single-letter names except loop counters (`i`, `j`) and generics (`T`, `U`) | 🟡 |
| NM-06 | Booleans prefixed `is*`, `has*`, `can*` — `isActive`, `hasPermission`, `canEdit` | 🟡 |
| NM-07 | File name matches primary exported class/component — `UserService.ts` exports `UserService` | 🟡 |
| NM-08 | DB tables: `snake_case` plural. DB columns: `snake_case`. No camelCase in SQL schemas | 🟠 |
| NM-09 | REST URLs: lowercase, hyphens, plural nouns — `/product-categories` not `/ProductCategories` | 🟡 |
| NM-10 | Consistent convention within a project — never mix `created_at` and `createdAt` in same DB | 🟠 |

---

## 3. Constants & Enums

| Rule | Description | Severity |
|---|---|---|
| CN-01 | No magic numbers or unexplained string literals — use named constants | 🟠 |
| CN-02 | Finite sets of values use enums — `OrderStatus.PENDING` not `'pending'` | 🟠 |
| CN-03 | Constants grouped by domain — `AppConstants.API`, `AppConstants.Auth`, `AppConstants.Payment` | 🟡 |
| CN-04 | No hardcoded URLs, endpoint paths, or environment-specific values in source | 🔴 |
| CN-05 | No hardcoded credentials, API keys, tokens, or secrets anywhere in source | 🔴 |
| CN-06 | Numeric separators used for large numbers — `86_400_000` not `86400000` | 🟡 |

---

## 4. Error Handling

| Rule | Description | Severity |
|---|---|---|
| EH-01 | No swallowed exceptions — `catch (e) {}` or `catch (e) { return null; }` never acceptable | 🔴 |
| EH-02 | Caught errors enriched with context (resource ID, operation) before re-throwing | 🟠 |
| EH-03 | Domain-specific error classes used — not raw `new Error('failed')` | 🟡 |
| EH-04 | Every `async` function has `try/catch` or `.catch()` on the chain | 🟠 |
| EH-05 | No floating promises — every async call is awaited or explicitly handled | 🟠 |
| EH-06 | API error responses never include stack traces, DB query text, or file paths | 🔴 |
| EH-07 | Every `500` error response includes `requestId`/`traceId` for user reporting | 🟡 |
| EH-08 | Python: no bare `except:` — always catch specific exception types | 🟠 |

---

## 5. Logging

| Rule | Description | Severity |
|---|---|---|
| LG-01 | No `console.log`, `print()`, or `System.out.println()` in production — use structured logger | 🟡 |
| LG-02 | All logs are structured JSON: `timestamp`, `level`, `service`, `traceId`, `message` | 🟠 |
| LG-03 | No PII in logs — no email, phone, name, password, SSN, card number | 🔴 |
| LG-04 | No session IDs, JWT tokens, or API keys in any log statement | 🔴 |
| LG-05 | Log user IDs not user data — `userId: 'usr_123'` not `email: 'user@example.com'` | 🔴 |
| LG-06 | Correct log level: `error` = unexpected failure, `warn` = degraded, `info` = lifecycle event | 🟡 |
| LG-07 | `traceId` read from incoming request headers and included on every log line | 🟠 |
| LG-08 | Log level configurable per environment — `warn` in production, `debug` in development | 🟠 |
| LG-09 | Sentry (or equivalent APM) integrated in every frontend and mobile application | 🔴 |

---

## 6. Type Safety

| Rule | Description | Severity |
|---|---|---|
| TS-01 | TypeScript: `strict: true` in every `tsconfig.json` | 🔴 |
| TS-02 | No `any` type — use `unknown` with type guard or define a proper type | 🟠 |
| TS-03 | Explicit return types on all public/exported functions | 🟡 |
| TS-04 | No `as` type assertion without a prior type guard | 🟠 |
| TS-05 | Python: all function signatures have type hints — `mypy --strict` clean | 🟡 |
| TS-06 | C#: `<Nullable>enable</Nullable>` in every `.csproj` | 🟠 |
| TS-07 | Discriminated unions for variant types — not optional fields (`Result<T>` pattern) | 🟡 |

---

## 7. Performance

| Rule | Description | Severity |
|---|---|---|
| PF-01 | No N+1 queries — DB/API call inside a loop → batch query, `Promise.all`, or DataLoader | 🔴 |
| PF-02 | No `SELECT *` — always name the specific columns required | 🟠 |
| PF-03 | All list endpoints paginated — cursor-based for large datasets, never unbounded | 🔴 |
| PF-04 | Indexes on all columns in WHERE, ORDER BY, GROUP BY, and JOIN ON | 🔴 |
| PF-05 | Connection pooling — never new DB or HTTP client per request | 🔴 |
| PF-06 | `Promise.all([...])` for independent async operations — never sequential awaits | 🟠 |
| PF-07 | All external calls: explicit timeout + retry with exponential backoff + jitter | 🟠 |
| PF-08 | Cache expensive/repeated reads with TTL — Redis or in-memory with eviction | 🟠 |
| PF-09 | No blocking sync I/O (`readFileSync`, `execSync`) in async request handlers | 🟠 |
| PF-10 | Heavy CPU work offloaded to worker thread or job queue — not on event loop | 🟠 |
| PF-11 | Frontend: lazy-load routes (`React.lazy`); skeleton screens not full-page spinners | 🟡 |
| PF-12 | API response time SLA: ≤ 800ms for all Lambda and GraphQL endpoints | 🟠 |

---

## 8. Testing

| Rule | Description | Severity |
|---|---|---|
| TT-01 | Unit tests for all business logic — service, domain, and utility layers | 🔴 |
| TT-02 | Tests cover: happy path + error conditions + edge cases + boundary values | 🟠 |
| TT-03 | Every user story acceptance criterion covered by at least one test | 🟠 |
| TT-04 | Mocks used for all external deps — tests never call real APIs or DBs | 🟠 |
| TT-05 | Test data from factories (`faker-js`, `factory_boy`) — no hardcoded objects | 🟡 |
| TT-06 | Coverage threshold in CI: ≥ 80% services, ≥ 90% domain/business logic | 🟠 |
| TT-07 | SonarLint installed in IDE — no unresolved issues committed | 🟠 |
| TT-08 | SonarQube quality gate blocks CI merge on failure | 🟠 |

---

## 9. Comments & Documentation

| Rule | Description | Severity |
|---|---|---|
| DC-01 | Comments explain WHY not WHAT — never restate what the code does | 🟡 |
| DC-02 | Comment on its own line — never trailing after code | 🟡 |
| DC-03 | Every module/file has a brief header comment describing its purpose | 🟡 |
| DC-04 | No `// TODO` in production-bound commits — track in issue tracker | 🟠 |
| DC-05 | All public APIs have Swagger/OpenAPI documentation | 🟠 |
| DC-06 | Postman collection maintained and committed to `docs/` | 🟡 |

---

## 10. Language-Specific Rules

### JavaScript / TypeScript
| Rule | Description | Severity |
|---|---|---|
| JS-01 | `const` by default; `let` when reassignment needed; `var` never | 🟡 |
| JS-02 | Strict equality `===` always — `==` never | 🟡 |
| JS-03 | `async/await` over `.then()` chains | 🟡 |
| JS-04 | No string first arg to `setTimeout`/`setInterval` — function form only | 🔴 |
| JS-05 | `package-lock.json` committed and never deleted | 🟠 |
| JS-06 | Node version in `package.json` `engines` field and `.nvmrc` | 🟡 |
| JS-07 | Axios interceptors for common request params — not manual per-call | 🟡 |

### React / React Native
| Rule | Description | Severity |
|---|---|---|
| RN-01 | No inline styles — `StyleSheet.create()` or CSS classes only | 🟡 |
| RN-02 | `key` props use stable unique IDs — never array index on dynamic lists | 🟠 |
| RN-03 | All `useEffect` deps complete; cleanup returned where subscriptions opened | 🟠 |
| RN-04 | No raw RN primitives (`Text`, `View`) in screen/container files — use custom components | 🟠 |
| RN-05 | API calls via Redux/Zustand actions — never direct axios in screen component | 🟠 |

### Python
| Rule | Description | Severity |
|---|---|---|
| PY-01 | No bare `except:` — catch specific exception types | 🟠 |
| PY-02 | No mutable default arguments | 🟠 |
| PY-03 | `with` statement for all file, DB, and connection handling | 🟠 |
| PY-04 | Pydantic models for all FastAPI request/response schemas | 🟠 |

### iOS / Swift
| Rule | Description | Severity |
|---|---|---|
| IOS-01 | No force unwrap `!` in production — use `guard let` or optional chaining | 🟠 |
| IOS-02 | Protocol conformance in separate `extension` with `// MARK: -` | 🟡 |
| IOS-03 | Delegate methods include delegate source as unnamed first parameter | 🟡 |
| IOS-04 | `let` by default; `var` only when mutation is required | 🟡 |

---

## 11. Instant Escalation — 🔴 Flag Immediately

| # | Violation |
|---|---|
| ESC-01 | PII (email, phone, password, SSN) in any log statement |
| ESC-02 | Swallowed exception: `catch (e) {}` |
| ESC-03 | `eval(userInput)` or `new Function(userInput)` |
| ESC-04 | OS `exec`/`system` with user-supplied input |
| ESC-05 | `Math.random()` for security token, OTP, or session ID |
| ESC-06 | No pagination on list endpoint that can return unbounded results |
| ESC-07 | N+1: DB/API call inside a loop with no batching |
| ESC-08 | Hardcoded secret, credential, or API key in source code |
| ESC-09 | `any` type on a security-critical data path |
