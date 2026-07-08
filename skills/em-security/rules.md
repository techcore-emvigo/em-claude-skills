# em-security — Rules Reference

Non-negotiable security rules enforced during every review and generation task.
Every violation is a finding — no exceptions, no "it's probably fine".

Prefix: **SC** = Secrets/Credentials | **IN** = Injection | **AU** = Authentication |
**AZ** = Authorization | **IV** = Input Validation | **SD** = Sensitive Data |
**CR** = Cryptography | **AP** = API/Communication | **SM** = Session Management |
**DP** = Data Protection | **FM** = File Management | **DB** = Database Security

---

## 1. Secrets & Credentials

| Rule | Description | Severity |
|---|---|---|
| SC-01 | No hardcoded secrets, API keys, tokens, or passwords anywhere in source code or config files | 🔴 |
| SC-02 | All secrets stored in a secrets manager — AWS Secrets Manager, Vault, GCP Secret Manager | 🔴 |
| SC-03 | No secrets in `.env` files committed to source control — `.env` always in `.gitignore` | 🔴 |
| SC-04 | No secrets in URL query parameters, log statements, or error messages | 🔴 |
| SC-05 | No secrets in Dockerfile `ENV` instructions | 🔴 |
| SC-06 | No secrets in CI/CD pipeline YAML files — use CI secrets store | 🔴 |
| SC-07 | Secrets validated at application startup — fail fast if any required secret is missing | 🟠 |
| SC-08 | Separate credentials per environment — dev/staging/prod never share the same keys | 🟠 |

---

## 2. Injection Prevention

| Rule | Description | Severity |
|---|---|---|
| IN-01 | Parameterised queries always — never string concatenation or f-string interpolation in SQL | 🔴 |
| IN-02 | NoSQL injection prevention — user input never passed directly as a query filter object | 🔴 |
| IN-03 | No `eval()`, `new Function()`, or `vm.runInContext()` with non-literal arguments | 🔴 |
| IN-04 | No OS shell commands built from user input — `exec`/`system`/`subprocess(shell=True)` | 🔴 |
| IN-05 | No `innerHTML`, `dangerouslySetInnerHTML`, or `v-html` with unsanitised user content | 🔴 |
| IN-06 | File paths from user input validated against a whitelist index map — no path traversal | 🔴 |
| IN-07 | Null bytes (`\x00`) and CRLF (`\r\n`) stripped before any string/path operation | 🟠 |
| IN-08 | URL parameters encoded with `encodeURIComponent` before use in URLs | 🟠 |
| IN-09 | Redirect URLs validated against an allowed-origins whitelist before `res.redirect()` | 🔴 |
| IN-10 | No `require(userInput)` or `import(userInput)` — module paths always static | 🔴 |

---

## 3. Authentication

| Rule | Description | Severity |
|---|---|---|
| AU-01 | Passwords hashed with `bcrypt` (cost ≥ 12) or `argon2id` — never MD5, SHA1, or SHA256 | 🔴 |
| AU-02 | Password hashing always server-side — never client-side | 🔴 |
| AU-03 | Auth failure message is identical for wrong username and wrong password — no enumeration | 🟠 |
| AU-04 | Account lockout after 5 consecutive failed login attempts with time-based cooldown | 🔴 |
| AU-05 | Password minimum 8 characters (16 recommended); must include uppercase, lowercase, digit | 🟠 |
| AU-06 | Password reset tokens: cryptographically random (`crypto.randomBytes(32)`), single-use, expire in 15 minutes | 🔴 |
| AU-07 | User notified by email on any password change | 🟠 |
| AU-08 | MFA enforced on all admin and high-privilege accounts | 🔴 |
| AU-09 | All vendor-supplied default passwords changed before deployment | 🔴 |
| AU-10 | JWT: `exp`, `iss`, `aud` claims validated on every request — `jwt.verify()` not `jwt.decode()` | 🔴 |
| AU-11 | Access token expiry ≤ 1 hour — never longer for private API access | 🔴 |
| AU-12 | No one-time tokens used as auth mechanism for regular private API access | 🔴 |
| AU-13 | Re-authentication required before critical operations (account deletion, payment, role change) | 🟠 |
| AU-14 | "Remember me" / persistent login disabled on sensitive forms | 🟠 |

---

## 4. Authorization

| Rule | Description | Severity |
|---|---|---|
| AZ-01 | Auth guard applied globally — nothing is unprotected by accident; `@Public()` for explicit opt-out | 🔴 |
| AZ-02 | Every POST/PUT/PATCH/DELETE endpoint requires authentication | 🔴 |
| AZ-03 | Role sourced from verified JWT payload only — never from request body, query, or header | 🔴 |
| AZ-04 | Ownership check on every resource fetch — verify `resource.userId === requestingUser.id` | 🔴 |
| AZ-05 | Return 404 (not 403) when an unauthorised user accesses a resource — do not confirm existence | 🟠 |
| AZ-06 | Auth check fails securely — `catch(e) { return false; }` never `return true` | 🔴 |
| AZ-07 | Admin/privileged logic in a separate service — never mixed with regular user logic | 🟠 |
| AZ-08 | `Referer` header used only as supplemental check — never as the sole authorisation mechanism | 🔴 |
| AZ-09 | Per-user transaction rate limits enforced to deter automated attacks | 🟠 |
| AZ-10 | Long-lived sessions re-validated periodically — revoke if role/status has changed | 🟠 |
| AZ-11 | Account disable immediately revokes all active sessions | 🔴 |
| AZ-12 | Unused accounts auto-disabled after 30 days of inactivity | 🟠 |
| AZ-13 | Service accounts and IAM roles use least privilege — no `Action: *` or `Resource: *` | 🔴 |

---

## 5. Input Validation

| Rule | Description | Severity |
|---|---|---|
| IV-01 | All validation on the server — client-side validation is UX only | 🔴 |
| IV-02 | Single centralised validation routine at every API boundary — not scattered per route | 🟠 |
| IV-03 | Validate type, length, range, format, and character set for every input field | 🟠 |
| IV-04 | Whitelist approach — allow only known-good characters; reject everything else | 🟠 |
| IV-05 | All validation failures result in rejection — never silently sanitise and continue | 🟠 |
| IV-06 | Query params, path params, headers, and cookies validated with the same rigour as body | 🟠 |
| IV-07 | Input truncated to maximum length before any copy/concatenation operation | 🟠 |
| IV-08 | UTF-8 normalised (canonicalised) before validation to prevent double-encoding bypass | 🟠 |
| IV-09 | CAPTCHA on all public registration, login, password-reset, and contact forms | 🟠 |

---

## 6. Sensitive Data Exposure

| Rule | Description | Severity |
|---|---|---|
| SD-01 | No PII in log statements — mask email, phone, SSN, DOB; log only user IDs | 🔴 |
| SD-02 | No passwords, tokens, or card data in any log statement — ever | 🔴 |
| SD-03 | Stack traces, DB query text, and file paths never returned in API error responses | 🟠 |
| SD-04 | User objects returned via API never include `passwordHash`, tokens, or internal IDs | 🔴 |
| SD-05 | Response DTOs explicitly whitelist fields — never return raw DB entities | 🟠 |
| SD-06 | Sensitive data in GET URL parameters forbidden — use POST body | 🔴 |
| SD-07 | PII not stored in cookies — session ID only (opaque token, not user data) | 🔴 |
| SD-08 | PII not forwarded as request headers downstream | 🔴 |
| SD-09 | Highly sensitive fields (SSN, DOB, card number) encrypted at the application layer before DB storage | 🔴 |
| SD-10 | Sensitive page cache disabled — `Cache-Control: no-cache, no-store, must-revalidate` on auth routes | 🟠 |
| SD-11 | Autocomplete disabled on sensitive form fields (password, card, SSN) | 🟠 |
| SD-12 | GDPR/data erasure supported — mechanism to anonymise PII on request | 🟠 |

---

## 7. Cryptography

| Rule | Description | Severity |
|---|---|---|
| CR-01 | `crypto.randomBytes(32)` / `secrets.token_bytes()` for all random tokens — never `Math.random()` | 🔴 |
| CR-02 | AES-256-GCM with random IV per encryption — never reuse IV | 🔴 |
| CR-03 | Timing-safe comparison (`crypto.timingSafeEqual`) for all secret/signature comparisons | 🟠 |
| CR-04 | Never use MD5, SHA1, DES, RC4 for any security purpose | 🔴 |
| CR-05 | All cryptographic operations server-side — never in client-side code | 🔴 |
| CR-06 | Encryption keys stored in secrets manager — never hardcoded or in config files | 🔴 |

---

## 8. API & Communication Security

| Rule | Description | Severity |
|---|---|---|
| AP-01 | `helmet()` applied on every HTTP service — sets CSP, X-Frame-Options, X-Content-Type-Options, HSTS | 🟠 |
| AP-02 | CORS: explicit origin whitelist — never `Access-Control-Allow-Origin: *` on authenticated APIs | 🔴 |
| AP-03 | Rate limiting on all auth endpoints (login, register, OTP, password-reset) | 🔴 |
| AP-04 | Rate limiting on all public endpoints | 🟠 |
| AP-05 | HTTPS enforced — HTTP redirects to HTTPS; `rejectUnauthorized: true` on all outbound clients | 🔴 |
| AP-06 | TLS minimum version `TLSv1.2` — TLS 1.0 and 1.1 disabled | 🔴 |
| AP-07 | Third-party CDN scripts use SRI `integrity=` hash — never plain external `<script src>` | 🟠 |
| AP-08 | Webhook signature verified against raw body using `timingSafeEqual` before any processing | 🔴 |
| AP-09 | Webhook handler returns `200 OK` within 5 seconds — processes payload asynchronously | 🔴 |
| AP-10 | GraphQL: depth limit + complexity limit configured; introspection auth-gated in production | 🔴 |
| AP-11 | gRPC: TLS on all connections; deadline set on every client call | 🔴 |
| AP-12 | Message consumers: idempotency check before processing — duplicates cause data corruption | 🔴 |
| AP-13 | DLQ configured on every async consumer — failed messages never silently discarded | 🔴 |
| AP-14 | `X-Powered-By` and `Server` headers removed from all HTTP responses | 🟡 |

---

## 9. Session Management

| Rule | Description | Severity |
|---|---|---|
| SM-01 | Session IDs cryptographically random — `crypto.randomBytes(32)` minimum | 🔴 |
| SM-02 | Session IDs never in URLs, logs, or error messages — HTTP cookie header only | 🔴 |
| SM-03 | Session cookies: `httpOnly: true`, `secure: true`, `sameSite: 'strict'` | 🔴 |
| SM-04 | Session inactivity timeout ≤ 2 hours | 🟠 |
| SM-05 | Full server-side session destroyed on logout — not only cookie cleared | 🔴 |
| SM-06 | New session ID generated after successful login (session fixation prevention) | 🟠 |
| SM-07 | CSRF token on all cookie-authenticated mutation endpoints | 🟠 |
| SM-08 | Pre-login session destroyed and new session created after authentication | 🟠 |

---

## 10. Data Protection & TLS

| Rule | Description | Severity |
|---|---|---|
| DP-01 | TLS certificate: valid domain, not expired, with intermediate chain | 🟠 |
| DP-02 | Failed TLS connection never falls back to HTTP — hard fail | 🔴 |
| DP-03 | `Content-Type` header includes `; charset=utf-8` on all responses | 🟡 |
| DP-04 | Sensitive URL parameters filtered from HTTP `Referer` header before external redirects | 🟠 |
| DP-05 | Server version information not disclosed in response headers | 🟡 |
| DP-06 | Directory listings disabled on all web servers | 🟠 |
| DP-07 | `robots.txt` disallows sensitive paths under one parent directory | 🟡 |
| DP-08 | Test code and debug routes removed before production deployment | 🔴 |
| DP-09 | Development environment isolated from production network | 🔴 |

---

## 11. Database Security

| Rule | Description | Severity |
|---|---|---|
| DB-01 | Application DB user: DML only (SELECT, INSERT, UPDATE, DELETE) — no DDL | 🔴 |
| DB-02 | Separate DB credentials per trust level: app, read-only, migration, admin | 🟠 |
| DB-03 | DB connection string from secrets manager — never hardcoded | 🔴 |
| DB-04 | DB port not exposed to public internet — VPC/private network only | 🔴 |
| DB-05 | All default DB vendor accounts changed or disabled | 🔴 |
| DB-06 | Unnecessary DB features, stored procedures, and extensions disabled | 🟠 |
| DB-07 | DB connections closed promptly — connection pool manages lifecycle | 🟠 |

---

## 12. File Management

| Rule | Description | Severity |
|---|---|---|
| FM-01 | Uploaded file type verified by magic bytes — extension check alone is insufficient | 🔴 |
| FM-02 | File size limit enforced on all upload endpoints | 🟠 |
| FM-03 | Server-generated safe filename used — never user-supplied `originalname` | 🔴 |
| FM-04 | Files stored outside web root — S3/Cloud Storage, never `public/uploads/` | 🔴 |
| FM-05 | Uploaded files scanned for malware before storage | 🟠 |
| FM-06 | Signed URL with short expiry returned to client — never absolute server file path | 🟠 |
| FM-07 | Execution privileges disabled on upload directories | 🔴 |
| FM-08 | User-supplied file paths mapped via whitelist index — no dynamic `path.join(base, userInput)` | 🔴 |
| FM-09 | Upload endpoint requires authentication | 🔴 |

---

## 13. Mobile Security

| Rule | Description | Severity |
|---|---|---|
| MB-01 | Sensitive data (tokens, PIN, password) stored in Keychain/Keystore — never `UserDefaults`/`SharedPreferences` | 🔴 |
| MB-02 | HTTPS enforced — `http://` URLs forbidden in any network configuration | 🔴 |
| MB-03 | ATS never disabled — `NSAllowsArbitraryLoads = true` blocked | 🔴 |
| MB-04 | SSL/certificate pinning implemented for production builds | 🟠 |
| MB-05 | No API keys, secrets, or base URLs hardcoded in mobile bundle | 🔴 |
| MB-06 | Base URL and environment config in remote config — not compiled into the app | 🟠 |
| MB-07 | Code obfuscation enabled — ProGuard/R8 (Android), Swift obfuscation (iOS) | 🔴 |
| MB-08 | App permissions minimised to only what the feature actually requires | 🔴 |
| MB-09 | Payment SDK (Stripe, GooglePay) called from backend service only — never from mobile screen | 🔴 |
| MB-10 | Card numbers, CVVs, and bank account data never logged | 🔴 |

---

## 14. Instant Escalation — 🔴 Flag Immediately

| # | Violation |
|---|---|
| ESC-01 | Any credential, token, or API key literal in source code or config |
| ESC-02 | User input reaches SQL, shell, `eval()`, or `innerHTML` without sanitisation |
| ESC-03 | Unauthenticated state-changing endpoint (POST/PUT/PATCH/DELETE) |
| ESC-04 | PII or secrets in any log statement |
| ESC-05 | Auth check fails open: `catch(e) { return true; }` |
| ESC-06 | IDOR: resource fetched by ID with no ownership or role check |
| ESC-07 | Webhook missing signature verification |
| ESC-08 | Message consumer missing idempotency — duplicates corrupt data |
| ESC-09 | No DLQ on async consumer — failures silently lost |
| ESC-10 | File type checked by extension only — no magic byte validation |
| ESC-11 | Files stored in web root — directly HTTP-accessible |
| ESC-12 | `rejectUnauthorized: false` on any outbound HTTPS client |
| ESC-13 | Sensitive data (SSN, DOB, card number) stored unencrypted in DB |
| ESC-14 | iOS `NSAllowsArbitraryLoads = true` in any build |
| ESC-15 | iOS sensitive data in `UserDefaults` instead of Keychain |
| ESC-16 | Payment SDK called directly from mobile screen — secret key in bundle |
| ESC-17 | Card/bank data in any log statement (PCI violation) |
