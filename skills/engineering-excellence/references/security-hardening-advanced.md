# Security Hardening — Advanced Controls

Covers: one-time token rules, no PII in request headers via cookie,
no third-party CDN without SRI, OWASP Top 10 quick reference,
cache-control headers for sensitive pages.

---

## 9. One-Time Token Rules for Private APIs

```typescript
// ❌ Bad — one-time-use tokens used for authenticated private API access
// OTP / magic links are for specific flows (email verify, password reset) ONLY
// Using a one-time token for a private API means:
// - Token expires after one call — user must re-authenticate constantly
// - Replay attacks if token is intercepted before first use
// - No revocation mechanism for multi-step flows

// ✅ Good — short-lived JWT with 1 hour expiry for private API access
const ACCESS_TOKEN_TTL  = '1h';    // 1 hour maximum for access tokens
const REFRESH_TOKEN_TTL = '7d';    // refresh token for obtaining new access tokens

// ✅ Good — one-time tokens ONLY for these specific flows:
// 1. Email verification link
// 2. Password reset link
// 3. Magic-link login (where explicitly intended)
// Store used tokens to prevent replay:
async function consumeOtpToken(token: string): Promise<boolean> {
  const key = `otp:used:${token}`;
  const alreadyUsed = !(await redis.set(key, '1', 'NX', 'EX', 3600));
  if (alreadyUsed) throw new UnauthorizedException('Token already used');
  return true;
}
```

---

## 10. Don't Pass PII Through Request Headers via Cookie

```typescript
// ❌ Bad — PII stored in cookie and sent with every request header
res.cookie('userData', JSON.stringify({
  email:  user.email,    // PII in cookie — sent in every request header
  name:   user.name,     // PII in cookie
  phone:  user.phone,    // PII in cookie
  role:   user.role,
}));

// ❌ Bad — accessing PII from cookie in middleware and passing downstream
app.use((req, res, next) => {
  const userData = JSON.parse(req.cookies.userData);
  req.headers['x-user-email'] = userData.email;  // PII propagated in headers
  next();
});

// ✅ Good — cookie contains only an opaque session ID
res.cookie('sessionId', session.id, {
  httpOnly: true,   // not accessible by JavaScript
  secure:   true,   // HTTPS only
  sameSite: 'strict',
  maxAge:   3600000,
});

// ✅ Good — server looks up session data server-side, never from cookie value
app.use(async (req, res, next) => {
  const sessionId = req.cookies.sessionId;
  if (sessionId) {
    req.user = await sessionStore.get(sessionId); // PII stays server-side
  }
  next();
});

// ✅ Good — JWT in Authorization header (not cookie) contains only non-sensitive claims
// JWT payload: { sub: 'usr_123', role: 'member', exp: ... }
// NOT: { email: '...', phone: '...', address: '...' }
```

---

## 11. No Direct Third-Party CDN URLs in HTML

```html
<!-- ❌ Bad — your app breaks if the third-party CDN goes down -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<link href="https://fonts.googleapis.com/css2?family=Inter" rel="stylesheet" />
<script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>

<!-- ✅ Good Option 1 — self-host assets (bundle via npm) -->
<!-- Install as npm dependency, bundle with webpack/vite -->
<!-- import 'chart.js' in your JS bundle -->

<!-- ✅ Good Option 2 — use CDN with Subresource Integrity (SRI) hash -->
<!-- SRI ensures the file hasn't been tampered with; browser rejects if hash mismatch -->
<script
  src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.0/chart.umd.min.js"
  integrity="sha512-ZwR1/gSZM3ai6vCdI+LVF1zSq/5HznD3oD+sCoJrzXJ+yKGtkTZo+TX2YRQSDCrR+R3+BOeFgRGc/FHqBmCg=="
  crossorigin="anonymous"
  referrerpolicy="no-referrer">
</script>

<!-- ✅ Good Option 3 — proxy through your own CDN/origin -->
<!-- Serve third-party fonts/assets through your own CloudFront/CDN -->
```

---

## 12. OWASP Top 10 Quick Reference

Every release must be assessed against the OWASP Top 10:
https://owasp.org/www-project-top-ten/

| # | Vulnerability | Key Prevention in Code |
|---|---|---|
| A01 | Broken Access Control | Ownership checks on every resource; deny-by-default |
| A02 | Cryptographic Failures | bcrypt/argon2 for passwords; HTTPS always; encrypt PII at rest |
| A03 | Injection | Parameterized queries; Zod/class-validator input validation |
| A04 | Insecure Design | Threat model; rate limiting; security in design phase |
| A05 | Security Misconfiguration | Helmet; CORS whitelist; no debug in production |
| A06 | Vulnerable Components | `npm audit`; Snyk; update dependencies every 6 months |
| A07 | Authentication Failures | JWT expiry 1h; HttpOnly cookies; MFA on admin |
| A08 | Data Integrity Failures | SRI hashes; signed releases; webhook signature verification |
| A09 | Logging Failures | Structured logs; audit trail; no PII in logs |
| A10 | SSRF | Validate URLs before server-side fetch; allowlist internal services |

---

## 13. Cache-Control & Pragma Headers

Prevent browsers and proxies from caching sensitive pages:

```typescript
// ✅ Good — prevent caching on authenticated/sensitive API responses
app.use('/api', (req, res, next) => {
  res.setHeader('Cache-Control', 'no-cache, no-store, must-revalidate');
  res.setHeader('Pragma',        'no-cache');
  res.setHeader('Expires',       '0');
  next();
});

// ✅ Good — allow caching on public, non-sensitive GET endpoints
app.get('/public/products', (req, res) => {
  res.setHeader('Cache-Control', 'public, max-age=300, stale-while-revalidate=60');
  res.json(products);
});
```

---

## Gap Detection Table (Additions)

| Gap | What to Look For | Severity |
|---|---|---|
| One-time token for private API access | Using OTP/magic-link tokens as the auth mechanism for regular APIs | 🔴 |
| PII in cookie value | `email`, `name`, `phone` in cookie string | 🔴 |
| Cookie PII forwarded as header | `req.headers['x-user-email'] = cookie.email` | 🔴 |
| Direct CDN URLs with no SRI | `<script src="https://cdn.../lib.js">` without `integrity=` attribute | 🟠 |
| No cache-control on sensitive API | Auth-required endpoints without `no-store` header | 🟠 |
| API token expiry > 1 hour | `expiresIn: '24h'` or `expiresIn: '7d'` on access token | 🔴 |
| No OWASP review before release | No documented security review against OWASP Top 10 | 🟠 |

---

