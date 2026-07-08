# Authentication — Session & Transmission Controls

Covers: authentication for external connections, secure password transmission,
session rotation, per-request CSRF tokens, redirect/open-redirect validation.

---

## 11. Authentication for External System Connections

```typescript
// ❌ Bad — credentials for external systems stored in source code
const dbClient  = new DatabaseClient({ password: 'realpassword' });
const s3Client  = new S3Client({ accessKeyId: 'AKIAIOSFODNN7EXAMPLE' });
const smtpConfig = { user: 'noreply@app.com', pass: 'smtp-password-123' };

// ❌ Bad — credentials passed as plain environment variables committed in .env
// .env file committed to git:
// SMTP_PASSWORD=actualpassword

// ✅ Good — all external credentials from secrets manager
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

class ExternalCredentialService {
  private cache = new Map<string, { value: string; fetchedAt: number }>();
  private TTL_MS = 5 * 60 * 1000; // refresh every 5 minutes

  async get(secretName: string): Promise<string> {
    const cached = this.cache.get(secretName);
    if (cached && Date.now() - cached.fetchedAt < this.TTL_MS) {
      return cached.value;
    }

    const client  = new SecretsManagerClient({ region: process.env.AWS_REGION });
    const command = new GetSecretValueCommand({ SecretId: secretName });
    const result  = await client.send(command);
    const value   = result.SecretString!;

    this.cache.set(secretName, { value, fetchedAt: Date.now() });
    return value;
  }
}

// ✅ Usage — credentials fetched at runtime, not build time
async function createSmtpTransport() {
  const password = await credentials.get('prod/smtp/password');
  return nodemailer.createTransport({
    host: process.env.SMTP_HOST,
    auth: { user: process.env.SMTP_USER, pass: password },  // secret from vault
  });
}
```

---


---

## 12. Secure Password Transmission & Storage Rules

```typescript
// ❌ Bad — password sent via GET request (appears in URL, browser history, logs)
app.get('/login', (req, res) => {
  const { username, password } = req.query; // URL: /login?password=secret123
});

// ❌ Bad — password in any log
logger.info('Login attempt', { username, password }); // NEVER

// ✅ Good — password only via POST over HTTPS
// POST /auth/login   Content-Type: application/json
// { "email": "user@example.com", "password": "..." }
// HTTPS encrypts the body in transit — GET exposes it in URL

// ✅ Good — only send temporary passwords over encrypted email
async function sendTemporaryPassword(email: string, tempPassword: string): Promise<void> {
  // Temp password sent ONLY via email to pre-registered address
  // Email must be sent over TLS (STARTTLS or SMTPS)
  await mailer.sendSecure({
    to:      email,  // only the pre-registered address — never to user-supplied email
    subject: 'Your temporary password',
    text:    `Your temporary password is: ${tempPassword}\n\nThis will expire in 15 minutes and must be changed on first use.`,
    // Note: never log tempPassword — it's in the email, not in logs
  });
  logger.info('Temporary password email sent', { maskedEmail: mask.email(email) });
  // Log the email (masked), not the password
}

// ✅ Password storage — write-only table pattern
// The passwords table/collection grants INSERT + UPDATE to the app user
// but SELECT only via a stored procedure that returns boolean (valid/invalid)
// The raw hash is never returned to the application layer directly
```

---


---

## 13. Session Rotation & Periodic Re-Keying

```typescript
// ✅ Good — rotate session ID periodically (mitigates session hijacking)
const SESSION_ROTATION_INTERVAL_MS = 20 * 60 * 1000; // rotate every 20 minutes

async function rotateSessionIfNeeded(req: Request, res: Response): Promise<void> {
  const session = req.session;
  if (!session) return;

  const ageMs = Date.now() - session.createdAt.getTime();
  if (ageMs > SESSION_ROTATION_INTERVAL_MS) {
    const oldId  = session.id;
    const newId  = generateSessionId();
    const data   = await sessionStore.get(oldId);

    // Atomic: write new session, delete old
    await sessionStore.set(newId, { ...data, createdAt: new Date() });
    await sessionStore.destroy(oldId);

    // Set new cookie — old cookie becomes invalid
    res.cookie('sessionId', newId, SESSION_COOKIE_OPTIONS);
    logger.debug('Session rotated', { userId: data.userId });
  }
}

// ✅ Good — generate new session ID when upgrading from HTTP to HTTPS
// Always use HTTPS in production — avoid the HTTP→HTTPS scenario entirely
// If unavoidable (e.g. behind a proxy), regenerate session at the boundary:
async function upgradeSessionOnHttpsTransition(req: Request, res: Response): Promise<void> {
  if (req.headers['x-forwarded-proto'] === 'https' && req.session?.createdOnHttp) {
    await regenerateSession(req, res);
    logger.info('Session regenerated on HTTP→HTTPS upgrade', { userId: req.user?.id });
  }
}
```

---


---

## 14. Sensitive Operation Tokens (CSRF + Per-Request Tokens)

```typescript
// ✅ Good — per-session CSRF token for account management operations
// Generated on login, stored in session, validated on every state-changing request

// ✅ Good — per-REQUEST token for highly critical operations (fund transfer, account delete)
// This is stronger than CSRF — a new token is required for each individual operation

class SecureOperationTokenService {

  // Generate a single-use token for one specific critical operation
  async issueOperationToken(
    userId:    string,
    operation: string, // e.g. 'TRANSFER_FUNDS', 'DELETE_ACCOUNT'
    ttlSecs:   number = 300, // 5 minutes
  ): Promise<string> {
    const token = crypto.randomBytes(32).toString('hex');
    const key   = `op_token:${userId}:${operation}:${token}`;

    await redis.set(key, '1', 'EX', ttlSecs); // single-use, expires
    return token;
  }

  async verifyAndConsume(userId: string, operation: string, token: string): Promise<void> {
    const key     = `op_token:${userId}:${operation}:${token}`;
    const deleted = await redis.del(key); // atomic delete — consumes token

    if (deleted === 0) {
      throw new SecurityError(
        'INVALID_OPERATION_TOKEN',
        'Operation token is invalid, expired, or already used.'
      );
    }
  }
}

// ✅ Usage — critical fund transfer requires a fresh per-request token
@Post('banking/transfer')
@UseGuards(JwtAuthGuard)
async initiateTransfer(
  @Body()    dto:            TransferDto,
  @Headers('x-operation-token') operationToken: string,
  @CurrentUser() user:      AuthUser,
): Promise<TransferResult> {
  // Verify the per-request token — prevents CSRF and replay attacks
  await this.operationTokenService.verifyAndConsume(user.id, 'TRANSFER_FUNDS', operationToken);
  return this.bankingService.transfer(dto, user.id);
}
```

---


---

## 15. Redirect Validation (Open Redirect Prevention)

```typescript
// ❌ Critical — open redirect: attacker sends users to malicious site
app.get('/logout', (req, res) => {
  res.redirect(req.query.returnUrl as string); // unvalidated redirect!
});
// Attack URL: /logout?returnUrl=https://evil.com/phishing

// ❌ Bad — relative path check is bypassable
const url = req.query.returnUrl as string;
if (url.startsWith('/')) res.redirect(url); // /\evil.com or //evil.com bypasses this!

// ✅ Good — whitelist of allowed redirect origins
const ALLOWED_REDIRECT_ORIGINS = (process.env.ALLOWED_REDIRECT_ORIGINS ?? '')
  .split(',')
  .map(s => s.trim())
  .filter(Boolean);

function isSafeRedirectUrl(url: string): boolean {
  try {
    const parsed = new URL(url, process.env.APP_BASE_URL); // resolve relative URLs

    // Allow same-origin redirects (same host)
    const appOrigin = new URL(process.env.APP_BASE_URL).origin;
    if (parsed.origin === appOrigin) return true;

    // Allow explicitly whitelisted external origins
    return ALLOWED_REDIRECT_ORIGINS.some(origin => parsed.origin === origin);
  } catch {
    return false; // invalid URL — reject
  }
}

// ✅ Middleware wrapper — safe for use on any redirect
function safeRedirect(res: Response, url: string, fallback = '/'): void {
  if (isSafeRedirectUrl(url)) {
    res.redirect(url);
  } else {
    logger.warn('Open redirect attempt blocked', { attemptedUrl: url });
    res.redirect(fallback); // fall back to safe default
  }
}

// ✅ Usage
app.get('/oauth/callback', (req, res) => {
  const returnUrl = req.query.state as string; // state was set before redirect
  safeRedirect(res, returnUrl, '/dashboard');
});
```

---
