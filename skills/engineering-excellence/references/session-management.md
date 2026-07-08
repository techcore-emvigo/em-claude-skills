# Session Management, Cryptography & Secure HTTP

Covers: session ID generation, cookie security attributes, inactivity timeout,
logout termination, CSRF, session rotation, crypto practices, HTTPS enforcement.

---

## 4. Session Management

### Session Creation — Server-Side Only

```typescript
// ❌ Bad — session ID generated client-side or predictably
const sessionId = Date.now().toString(); // predictable!
const sessionId = user.id + '-session';  // guessable!

// ✅ Good — cryptographically random session ID, server-side only
import crypto from 'crypto';

function generateSessionId(): string {
  return crypto.randomBytes(32).toString('hex'); // 256 bits of entropy
}

// ✅ Good — complete session creation on login
async function createSession(userId: string, req: Request): Promise<string> {
  // If a pre-login session exists, destroy it first
  if (req.session?.id) {
    await sessionStore.destroy(req.session.id);
  }

  const sessionId  = generateSessionId();
  const expiresAt  = new Date(Date.now() + SESSION_DURATION_MS);

  await sessionStore.set(sessionId, {
    userId,
    createdAt:  new Date(),
    expiresAt,
    ipAddress:  req.ip,
    userAgent:  req.headers['user-agent'],
    lastActive: new Date(),
  });

  return sessionId;
}
```

### Cookie Security Attributes

```typescript
// ✅ Good — secure cookie configuration for session tokens
const SESSION_COOKIE_OPTIONS: CookieOptions = {
  httpOnly: true,     // JavaScript cannot access this cookie — prevents XSS token theft
  secure:   true,     // HTTPS only — never sent over HTTP
  sameSite: 'strict', // not sent on cross-site requests — CSRF protection
  path:     '/',
  maxAge:   SESSION_DURATION_MS,
  // domain: '.yourdomain.com'  — set explicitly in production
};

res.cookie('sessionId', sessionId, SESSION_COOKIE_OPTIONS);

// ❌ Bad cookie configurations:
res.cookie('sessionId', id, { httpOnly: false });  // XSS can steal token
res.cookie('sessionId', id, { secure: false });    // sent over HTTP — sniffable
res.cookie('sessionId', id, { sameSite: 'none', secure: false }); // CSRF risk

// ✅ Good — DO NOT expose session IDs in URLs or logs
// ❌ Bad: GET /profile?sessionId=abc123  — session exposed in browser history and logs
// ❌ Bad: logger.info('Session created', { sessionId }) — session in logs
```

### Session Inactivity Timeout

```typescript
// ✅ Good — session timeout middleware
const SESSION_INACTIVITY_TIMEOUT_MS = 2 * 60 * 60 * 1000; // 2 hours

async function sessionTimeoutMiddleware(req: Request, res: Response, next: NextFunction) {
  const sessionId = req.cookies.sessionId;
  if (!sessionId) return next();

  const session = await sessionStore.get(sessionId);
  if (!session) {
    res.clearCookie('sessionId');
    return res.status(401).json({ error: { code: 'SESSION_EXPIRED' } });
  }

  const inactiveMs = Date.now() - session.lastActive.getTime();
  if (inactiveMs > SESSION_INACTIVITY_TIMEOUT_MS) {
    await sessionStore.destroy(sessionId);
    res.clearCookie('sessionId');
    return res.status(401).json({ error: { code: 'SESSION_TIMEOUT' } });
  }

  // Refresh last-active timestamp
  await sessionStore.touch(sessionId, { lastActive: new Date() });
  req.user = session.userId;
  next();
}
```

### Session Logout — Full Termination

```typescript
// ❌ Bad — cookie cleared client-side but session still active on server
app.post('/logout', (req, res) => {
  res.clearCookie('sessionId');  // server session still valid — can be replayed!
  res.json({ success: true });
});

// ✅ Good — destroy server-side session AND clear cookie
app.post('/logout', async (req, res) => {
  const sessionId = req.cookies.sessionId;

  if (sessionId) {
    await sessionStore.destroy(sessionId);  // server-side termination
    logger.info('User logged out', { userId: req.user?.id });
  }

  res.clearCookie('sessionId', {
    httpOnly: true,
    secure:   true,
    sameSite: 'strict',
    path:     '/',
  });

  res.json({ success: true });
});
```

### CSRF Protection

```typescript
// ✅ Good — CSRF token for cookie-based session auth (SameSite=Strict helps but isn't enough alone)
import csrf from 'csurf';

// CSRF middleware — generates token and validates on mutations
const csrfProtection = csrf({ cookie: { httpOnly: true, secure: true } });

// Apply to all state-changing routes with cookie auth
app.use('/api', csrfProtection);

// Frontend must include the CSRF token in the X-CSRF-Token header
// <meta name="csrf-token" content="{{ csrfToken }}" />
// axios.defaults.headers.common['X-CSRF-Token'] = getCsrfToken();

// ✅ Good — per-request token for highly sensitive operations
@Post('transfer')
@UseGuards(JwtAuthGuard)
async transferFunds(
  @Headers('x-request-token') requestToken: string,
  @CurrentUser() user: AuthUser,
  @Body() dto: TransferDto,
): Promise<void> {
  // Verify single-use per-request token for critical operation
  await this.tokenService.verifyAndConsumeRequestToken(user.id, requestToken);
  await this.bankingService.transfer(dto);
}
```

### Prevent Concurrent Logins (Optional — High-Security Apps)

```typescript
// ✅ Good — invalidate old session when new login occurs (single session policy)
async function loginWithSingleSession(userId: string): Promise<string> {
  // Revoke all existing sessions for this user
  await sessionStore.revokeAllForUser(userId);

  // Create fresh session
  const sessionId = await createSession(userId);
  logger.info('Login: previous sessions revoked', { userId });
  return sessionId;
}
```

---


---

## 5. Cryptographic Practices

```typescript
// ❌ Critical — never use for security purposes
Math.random()         // predictable — not cryptographically random
Date.now()            // predictable — used for tokens? guessable!
Math.random().toString(36) // not for tokens, OTPs, or session IDs

// ✅ Good — cryptographically secure random values
import crypto from 'crypto';

// Secure random token (URL-safe)
const token = crypto.randomBytes(32).toString('hex');        // 64-char hex
const token = crypto.randomBytes(32).toString('base64url');  // shorter, URL-safe

// Secure OTP (numeric)
const otp = crypto.randomInt(100000, 999999).toString();     // 6-digit OTP

// ✅ Good — timing-safe comparison (prevents timing attacks)
function timingSafeEqual(a: string, b: string): boolean {
  const bufA = Buffer.from(a);
  const bufB = Buffer.from(b);
  if (bufA.length !== bufB.length) return false;
  return crypto.timingSafeEqual(bufA, bufB);
}

// ✅ Good — AES-256-GCM encryption for data at rest
function encrypt(plaintext: string, keyHex: string): { iv: string; tag: string; ciphertext: string } {
  const key = Buffer.from(keyHex, 'hex');
  const iv  = crypto.randomBytes(12);  // unique IV per encryption — never reuse!
  const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);

  const encrypted = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  const tag       = cipher.getAuthTag();

  return {
    iv:         iv.toString('hex'),
    tag:        tag.toString('hex'),
    ciphertext: encrypted.toString('hex'),
  };
}
```

---


---

## 6. Secure HTTP Communication

```typescript
// ✅ Good — enforce HTTPS everywhere
import helmet from 'helmet';

app.use(helmet.hsts({
  maxAge:            365 * 24 * 60 * 60,  // 1 year in seconds
  includeSubDomains: true,
  preload:           true,
}));

// Redirect HTTP to HTTPS
app.use((req, res, next) => {
  if (!req.secure && process.env.NODE_ENV === 'production') {
    return res.redirect(301, `https://${req.headers.host}${req.url}`);
  }
  next();
});

// ✅ Good — only ASCII in response headers
function safeHeader(value: string): string {
  // Strip any non-ASCII characters from header values
  return value.replace(/[^\x20-\x7E]/g, '');
}

res.setHeader('X-User-Name', safeHeader(user.name)); // safe even if name has unicode
```

---
