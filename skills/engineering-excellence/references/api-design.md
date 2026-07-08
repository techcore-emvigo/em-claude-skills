# REST API — Gap Detection

## Design Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Verb in URL | `/getUsers`, `/createOrder`, `/deleteUser/123` | 🟡 |
| Wrong HTTP method | `GET` used to mutate state; `POST` used for idempotent fetch | 🟠 |
| Inconsistent naming | Mix of camelCase, snake_case, PascalCase in URL segments | 🟡 |
| Deep URL nesting | `/users/1/orders/2/items/3/reviews` — more than 2 levels | 🟡 |
| No versioning | No `/v1/` prefix or header — breaking changes impossible to manage | 🟠 |
| Internal IDs exposed | Sequential integer IDs in URLs (`/users/1`) — data volume disclosure | 🟡 |

## Security Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No auth on mutating endpoint | POST/PUT/PATCH/DELETE route without authentication | 🔴 |
| No ownership check | Fetching `/orders/{id}` without verifying the order belongs to the caller | 🔴 |
| Missing rate limiting | Login, register, OTP, password-reset endpoints with no throttle | 🔴 |
| CORS wildcard on authenticated API | `Access-Control-Allow-Origin: *` with cookie auth | 🔴 |
| Missing security headers | No `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security` | 🟠 |
| Stack trace in error response | `{ error: err.stack }` returned to client | 🟠 |
| Missing input validation | Request body used without schema validation | 🟠 |

## Response Quality Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Always 200 OK | Errors returned with `200` status and error flag in body | 🟠 |
| No consistent error envelope | Error responses differ per endpoint (no shared shape) | 🟡 |
| No `request_id` in response | Errors with no traceability ID — impossible to debug in production | 🟡 |
| No pagination on list endpoint | List endpoint returns all records with no page/cursor | 🔴 |
| Full entity returned | Response includes sensitive or irrelevant fields from DB entity | 🟠 |
| No `Location` header on 201 | `POST` that creates a resource returns no `Location: /resources/{id}` | 🟡 |

## Generation Checklist
- [ ] Resources are nouns, plural: `/users`, `/orders`
- [ ] HTTP methods match intent: GET (read), POST (create), PUT/PATCH (update), DELETE (remove)
- [ ] Auth middleware applied at router level, not per-route
- [ ] Row-level ownership verified for every resource fetch/mutation
- [ ] Consistent error envelope: `{ error: { code, message, request_id } }`
- [ ] All list endpoints paginated with cursor or page params
- [ ] Response DTOs filter sensitive fields before returning
