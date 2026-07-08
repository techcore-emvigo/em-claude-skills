# Security Gap Checklist — Core

Scan code for each category. Every match is a finding.
Covers: secrets, injection, auth/authz, input validation, sensitive data,
crypto, API/communication, async/queue, infrastructure.

---

# Security Gap Checklist

Use this file during every review. Scan code for each category below.
Mark every match as a finding — no exceptions, no "it's probably fine".

---

## 1. Secrets & Credentials

| Gap | What to Look For | Severity |
|---|---|---|
| Hardcoded secret | String literals matching: password, secret, key, token, api_key, private_key in source | 🔴 |
| Secret in config file | `.env`, `config.json`, `appsettings.json` committed with real values | 🔴 |
| Secret in log statement | `logger.info(password)`, `console.log(token)`, `print(api_key)` | 🔴 |
| Secret in URL | `https://user:pass@host`, `?api_key=real_value` | 🔴 |
| Secret in environment variable placeholder | `process.env.SECRET` used but no secrets manager reference | 🟠 |

**Fix pattern:** Always use a secrets manager (AWS Secrets Manager, Vault, GCP Secret Manager).
Never pass secrets as constructor args or function parameters that get logged.

---

## 2. Injection Vulnerabilities

| Gap | What to Look For | Severity |
|---|---|---|
| SQL injection | String concatenation into SQL: `"SELECT * FROM users WHERE id = " + userId` | 🔴 |
| NoSQL injection | Unvalidated object passed directly: `collection.find(req.body)` | 🔴 |
| Command injection | `exec(userInput)`, `spawn(userInput)`, `eval(userInput)` | 🔴 |
| XSS | `innerHTML = userInput`, `dangerouslySetInnerHTML={{ __html: userInput }}`, `v-html="userInput"` | 🔴 |
| Path traversal | `fs.readFile(userInput)`, `path.join(baseDir, req.params.file)` without validation | 🔴 |
| Template injection | Template strings built from user input: `\`Hello ${req.query.name}\`` in eval/templates | 🟠 |

**Fix pattern:** Parameterized queries, input validation + whitelist, DOMPurify for HTML, path.resolve + startsWith check for paths.

---

## 3. Authentication & Authorization

| Gap | What to Look For | Severity |
|---|---|---|
| Unprotected endpoint | POST/PUT/PATCH/DELETE route with no auth guard/middleware | 🔴 |
| Missing authorization check | Auth present but no ownership check: any user can access any resource by ID | 🔴 |
| JWT not verified | `jwt.decode()` used instead of `jwt.verify()` | 🔴 |
| JWT claims not validated | `exp`, `iss`, `aud` not checked after decode | 🔴 |
| Token in localStorage | `localStorage.setItem('token', ...)` — vulnerable to XSS | 🟠 |
| Weak session config | `httpOnly: false` on cookies, `secure: false` on cookies in prod | 🔴 |
| Missing CSRF protection | Stateful auth (cookies) without CSRF token on mutations | 🟠 |
| Privilege escalation | Role/permission check using user-supplied value: `if (req.body.role === 'admin')` | 🔴 |
| Missing auth on admin routes | `/admin/*` routes without elevated permission check | 🔴 |

**Fix pattern:** Auth middleware applied at router level, not per-route. Verify JWT signature + claims. Row-level ownership checks. Roles from the verified token, never from request body.

---

## 4. Input Validation

| Gap | What to Look For | Severity |
|---|---|---|
| No validation on request body | Controller/handler uses `req.body` fields directly without schema validation | 🟠 |
| No validation on query params | `req.query.page`, `req.params.id` used without type/range check | 🟠 |
| No validation on event payload | Queue/event consumer uses payload fields without schema check | 🟠 |
| Missing length limits | String fields with no max length — enables oversized payload attacks | 🟡 |
| Type coercion relied on | `parseInt(req.query.id)` without checking the result is a valid number | 🟡 |

**Fix pattern:** Validate at the boundary. Use Zod, Joi, class-validator, Pydantic, FluentValidation, or protobuf validators. Whitelist expected fields; reject unknown fields.

---

## 5. Sensitive Data Exposure

| Gap | What to Look For | Severity |
|---|---|---|
| PII in logs | `email`, `phone`, `ssn`, `dob`, `address` in log statements | 🔴 |
| Password hash in response | User object returned with `password` or `passwordHash` field | 🔴 |
| Stack trace in API response | `res.json(error)`, returning `error.stack` to client | 🟠 |
| Internal IDs in URLs | Sequential integer IDs expose data volume: `/users/1`, `/orders/42` | 🟡 |
| Verbose error messages | Error details that reveal DB schema, file paths, or internal logic | 🟠 |
| Missing response field filter | Returning full DB entity when only subset is needed | 🟡 |

**Fix pattern:** Response DTOs/serializers that explicitly include only safe fields. Sanitize error messages in production. Use UUIDs for public-facing IDs. Log user IDs not PII.

---

## 6. Cryptography & Hashing

| Gap | What to Look For | Severity |
|---|---|---|
| Weak password hashing | `md5(password)`, `sha1(password)`, `sha256(password)` for passwords | 🔴 |
| Missing salt | `hash(password)` without a unique salt per user | 🔴 |
| Weak random | `Math.random()` for tokens, OTPs, or session IDs | 🔴 |
| Hardcoded IV/salt | Reused IV in AES encryption | 🔴 |
| Outdated algorithm | DES, RC4, MD5, SHA1 for any security purpose | 🔴 |
| Plain HTTP | `http://` URLs for API calls, webhooks, or external service calls in production | 🟠 |

**Fix pattern:** `bcrypt`/`argon2` for passwords. `crypto.randomBytes(32)` for tokens. AES-256-GCM with random IV per encryption. HTTPS always.

---

## 7. API & Communication Security

| Gap | What to Look For | Severity |
|---|---|---|
| Missing rate limiting | Login, register, password-reset, OTP endpoints with no rate limit | 🔴 |
| Missing CORS restriction | `cors({ origin: '*' })` on authenticated APIs | 🟠 |
| Missing security headers | No `helmet()`, no CSP, no `X-Frame-Options`, no `Strict-Transport-Security` | 🟠 |
| Webhook without signature check | Webhook handler processes payload without verifying HMAC signature | 🔴 |
| Webhook processes before ACK | Heavy work done before returning 200 — causes retries + duplicates | 🔴 |
| GraphQL introspection public | `introspection: true` in production with no auth | 🟠 |
| GraphQL no depth limit | No `depthLimit()` rule — deeply nested query can DoS the server | 🔴 |
| GraphQL no complexity limit | No query complexity rule — expensive query can DoS the server | 🔴 |
| gRPC no deadline | gRPC client call with no deadline — can hang indefinitely | 🟠 |
| gRPC no TLS | Plain HTTP/2 (`insecure`) used in production | 🔴 |

---

## 8. Async & Queue Security

| Gap | What to Look For | Severity |
|---|---|---|
| No idempotency on consumer | Event/message processed without checking if already handled | 🔴 |
| No DLQ configured | Async consumer with no dead-letter queue — failed messages silently lost | 🔴 |
| No message schema validation | Consumer uses message fields without shape validation | 🟠 |
| Unauthenticated queue access | SQS/Kafka/RabbitMQ accessible without credentials or IAM | 🔴 |

---

## 9. Infrastructure & Secrets in Config

| Gap | What to Look For | Severity |
|---|---|---|
| Wildcard IAM / open security group | `0.0.0.0/0` ingress, `Action: "*"`, `Resource: "*"` | 🔴 |
| Public DB port | RDS, MongoDB, Redis port open to internet | 🔴 |
| Public S3 bucket | `BlockPublicAcls: false` without explicit justification | 🔴 |
| Secrets in Dockerfile ENV | `ENV API_KEY=real_value` in Dockerfile | 🔴 |
| Root container user | No `USER appuser` in Dockerfile — runs as root | 🟠 |
| Pinned image version missing | `FROM node:latest` — non-deterministic, potential supply chain risk | 🟡 |

---

