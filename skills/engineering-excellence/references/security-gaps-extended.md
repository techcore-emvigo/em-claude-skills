# Security Gap Checklist — Extended Part 1

Covers: pre-release security scanning, input validation detail, output encoding,
authentication/password, session management, access control.

---

# Security Gap Checklist — Extended

Extended gap tables for pre-release scanning, detailed auth/session/access
control, data protection, file upload, error logging, DB/config, dynamic exec.

---

## 10. Pre-Release Security Scanning

| Gap | What to Look For | Severity |
|---|---|---|
| MobSF scan not completed for mobile app | Mobile security vulnerabilities undetected before release | 🔴 |
| OWASP ZAP Proxy scan not run on web app | Web application attack vectors not validated | 🔴 |
| Nuclei scan not completed | Template-based vulnerability patterns missed | 🔴 |
| Dependency vulnerability scan not run (npm audit, pip audit, Snyk) | Known CVEs shipped in dependencies | 🔴 |
| Outdated or vulnerable libraries/plugins in release | Supply chain attack surface | 🔴 |
| Security patches not applied for dependencies | Known exploits unpatched | 🔴 |
| CSP (Content Security Policy) header not implemented | XSS attack vector open | 🔴 |
| Open port and attack surface scan not performed | Unnecessary exposure to internet | 🟠 |
| MFA not enforced for admin access | Admin account takeover risk | 🔴 |
| SonarQube security hotspots not reviewed | Code-level security issues in production | 🟠 |

**Tools to mandate in CI:**
- `npm audit` / `pip audit` / `bundle audit` — dependency CVE scanning
- Snyk / Dependabot — continuous dependency monitoring
- OWASP ZAP — dynamic application security testing
- MobSF — mobile application security framework
- Nuclei — template-based vulnerability scanner
- SonarQube/SonarCloud — static code analysis with security rules

---

## 11. Input Validation Gaps (Detailed)

| Gap | What to Look For | Severity |
|---|---|---|
| No centralised validator | Same validation logic copy-pasted across multiple controllers/handlers | 🟠 |
| Missing type validation | `req.query.limit` used as number with no `z.coerce.number()` or `parseInt` + `isNaN` check | 🟠 |
| Missing range validation | `age`, `quantity`, `price`, `amount` with no `.min()` / `.max()` bounds | 🟠 |
| Missing length limit | String field with no `.max()` — enables oversized payload / buffer attacks | 🟠 |
| No character whitelist | Free-text field accepts `<>"'%()&+\` with no escaping or rejection | 🟠 |
| Null byte not filtered | File path, DB query param, or search term without `\x00` check | 🔴 |
| CRLF injection not checked | Header-bound field without `\r\n` stripping | 🟠 |
| Path traversal not validated | `fs.readFile(req.params.file)` or `path.join(base, userInput)` without bounds check | 🔴 |
| Redirect URL unvalidated | `res.redirect(req.query.returnUrl)` with no whitelist check | 🔴 |
| `//evil.com` bypass | Redirect check `url.startsWith('/')` without parsing with `new URL()` | 🔴 |
| UTF-8 not canonicalised | Input validated before UTF-8 decode — double-encoding bypass possible | 🟠 |
| Headers not ASCII-validated | Non-ASCII characters allowed in HTTP header values | 🟠 |
| Cookie values not validated | Cookie names/values used directly without schema validation | 🟠 |
| POST-back from JS not validated | Automated JavaScript AJAX posts not routed through input validation | 🟠 |

---

## 12. Output Encoding Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| User data in HTML without encoding | `\`<div>${req.query.name}</div>\`` — raw interpolation | 🔴 |
| `innerHTML` with user content | `element.innerHTML = userInput` without DOMPurify | 🔴 |
| No `encodeURIComponent` on URL params | `\`/search?q=${userInput}\`` without encoding | 🟠 |
| Dynamic ORDER BY without whitelist | `\`ORDER BY ${req.query.sort}\`` without allowlist check | 🔴 |
| OS command with user input | `exec(\`convert ${userInput}\`)` — command injection | 🔴 |
| LDAP filter with unescaped input | LDAP query built from user data without escaping | 🔴 |
| XML/HTML entity injection | User data in XML/HTML template without entity encoding | 🟠 |

---

## 13. Authentication & Password Management Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| MD5 or SHA1 password hash | `md5(password)`, `sha1(password)`, `sha256(password)` for passwords | 🔴 |
| Client-side password hashing | Password hashed in browser JS before sending — server skips hashing | 🔴 |
| Unsalted hash | `crypto.createHash('sha256').update(password).digest('hex')` | 🔴 |
| bcrypt rounds < 12 | `bcrypt.hash(pwd, 10)` — 12 is minimum; 14 for high-security | 🟠 |
| Verbose auth error | Separate "Invalid username" vs "Invalid password" messages | 🟠 |
| No account lockout | Login endpoint with no failed-attempt counter or rate limit | 🔴 |
| Lockout too short | Lockout < 5 minutes — brute force still viable | 🟠 |
| Predictable reset token | `Math.random()`, `Date.now()`, or `userId` used for reset token | 🔴 |
| Reset token not single-use | Token not marked used after password reset | 🔴 |
| Reset token expiry > 15 min | Long-lived reset tokens increase hijack window | 🟠 |
| Reset email to unregistered address | Password reset sent to user-supplied email (not pre-registered) | 🔴 |
| No notification on password change | User not emailed when their password is reset | 🟠 |
| Password reuse allowed | No history check — same password reused immediately | 🟠 |
| No minimum password age | Password changed multiple times rapidly to cycle back to original | 🟡 |
| Temp password not forced changed | `isTemporaryPassword` flag set but not checked in middleware | 🔴 |
| No re-auth before critical action | Account deletion, payment, email change with no password confirmation | 🟠 |
| No MFA on admin accounts | Admin users authenticating with password only | 🔴 |
| Vendor default credentials active | Default `admin/admin`, `root/root`, `admin/password` still active | 🔴 |
| External credentials in source code | DB password, SMTP password, API key committed to repo | 🔴 |
| Password via GET request | Login credentials in URL query string | 🔴 |
| Credentials logged | `password`, `passwordHash`, `token`, `apiKey` in any log statement | 🔴 |
| No credential spray detection | No monitoring for same password tried against many accounts | 🟠 |
| Last login not shown to user | No "last login was X from Y" message at login | 🟡 |
| Weak security questions | "What's your favorite book?" — common, guessable answers | 🟠 |

---

## 14. Session Management Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Predictable session ID | `Date.now()`, `userId + timestamp`, sequential number | 🔴 |
| Session ID in URL | `?session=`, `?token=`, `?sid=` in any GET URL | 🔴 |
| Session ID in log | `logger.info({ sessionId })`, `console.log(session)` | 🔴 |
| Session ID in error message | Error response body contains session identifier | 🔴 |
| `httpOnly: false` on session cookie | Cookie accessible via `document.cookie` | 🔴 |
| `secure: false` on session cookie | Cookie sent over unencrypted HTTP | 🔴 |
| No `sameSite` on session cookie | Missing sameSite attribute — CSRF risk | 🟠 |
| No session inactivity timeout | Sessions live forever with no activity check | 🟠 |
| Session not destroyed on logout | Only cookie cleared — server session still valid | 🔴 |
| Pre-login session reused post-login | Session ID same before and after authentication | 🟠 |
| No session rotation | Session ID never rotated during active use | 🟡 |
| No CSRF token on cookie auth | State-changing endpoint with cookie auth, no CSRF check | 🟠 |
| No per-request token on critical ops | Fund transfer / account delete with only session-level token | 🟠 |
| Persistent login enabled | "Remember me" saves session indefinitely | 🟠 |
| Concurrent sessions unrestricted | High-security app allows same account from unlimited devices | 🟡 |
| No new session on HTTP→HTTPS transition | Session created on HTTP reused on HTTPS | 🟠 |

---

