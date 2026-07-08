# Authentication — Password Controls & Monitoring

Covers: password history enforcement, last-login reporting, credential spray
detection, temporary password enforcement, vendor credential rotation.

---

# Authentication — Advanced Controls

Covers: password history, last-login reporting, credential spray detection,
temporary password enforcement, vendor credential rotation, external system
auth, secure transmission, session rotation, CSRF tokens, redirect validation.

---

## Gap Detection Table

### Input Validation Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No server-side validation | Client validates but server trusts `req.body` directly | 🔴 |
| No centralised validator | Same validation logic repeated across multiple routes | 🟠 |
| Missing length limits | String field with no `.max()` constraint | 🟠 |
| Missing type validation | `req.query.page` used as number without `parseInt` + `isNaN` check | 🟠 |
| Missing range validation | Age, quantity, amount with no min/max bounds | 🟠 |
| No character whitelist | Free-text accepting `<>'"()%&+\` without escaping | 🟠 |
| Null byte not checked | File paths or DB queries with no null byte `\x00` check | 🔴 |
| CRLF not checked | Header values containing `\r\n` without sanitisation | 🟠 |
| Path traversal not checked | `../` in file paths or URL params without validation | 🔴 |
| Redirect URL not validated | `res.redirect(req.query.returnUrl)` without whitelist check | 🔴 |
| No UTF-8 normalisation | Unicode not canonicalised before validation — double-encoding bypass | 🟠 |

### Authentication Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| MD5 / SHA1 password hashing | `md5(password)`, `sha1(password)` | 🔴 |
| Unsalted hash | `sha256(password)` without unique salt | 🔴 |
| Client-side password hashing | Password hashed in JS before sending | 🔴 |
| Verbose auth error | `'Invalid password'` vs `'Invalid username'` — reveals which is wrong | 🟠 |
| No account lockout | Login endpoint with no failed-attempt counter | 🔴 |
| Weak reset token | `Math.random()` used for password reset token | 🔴 |
| Reset token not single-use | Token can be reused after password reset | 🔴 |
| No token expiry | Password reset or verification token with no expiry | 🔴 |
| User not notified on password change | No email sent when password is reset | 🟠 |
| No re-auth before critical action | Account deletion / payment without password confirmation | 🟠 |
| No MFA on admin accounts | Admin users have no MFA requirement | 🔴 |
| Default vendor credentials | Framework/DB/service still using default admin/password | 🔴 |
| Password reuse allowed | No history check on password change | 🟠 |

### Session Management Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Predictable session ID | `Date.now()` or `userId + timestamp` as session ID | 🔴 |
| Session ID in URL | `?sessionId=abc` or `?token=abc` in URL | 🔴 |
| Session ID in logs | `logger.info({ sessionId })` | 🔴 |
| `httpOnly: false` on session cookie | Cookie readable by JavaScript | 🔴 |
| `secure: false` on session cookie | Cookie sent over HTTP | 🔴 |
| No session inactivity timeout | Session never expires from inactivity | 🟠 |
| Server session not destroyed on logout | Only cookie cleared — old session still valid | 🔴 |
| No new session ID after login | Same session ID before and after authentication | 🟠 |
| No CSRF protection on cookie-auth | Cookie-auth mutations without CSRF token | 🟠 |
| Concurrent logins allowed | High-security app allows same user logged in from multiple devices | 🟡 |

---


---

## 8. Password History & Change Controls

```typescript
// ✅ Good — prevent password reuse (last 5 passwords)
class PasswordHistoryService {
  private readonly HISTORY_COUNT = 5;
  private readonly MIN_PASSWORD_AGE_HOURS = 24; // passwords must be 1 day old before changing

  async enforceHistory(userId: string, newPlainPassword: string): Promise<void> {
    const history = await this.passwordHistoryRepo.findRecent(userId, this.HISTORY_COUNT);

    for (const entry of history) {
      const isReused = await bcrypt.compare(newPlainPassword, entry.passwordHash);
      if (isReused) {
        throw new ValidationError(
          'PASSWORD_REUSED',
          `Password was used recently. Choose a password not used in the last ${this.HISTORY_COUNT} changes.`
        );
      }
    }
  }

  async enforceMinAge(userId: string): Promise<void> {
    const lastChange = await this.passwordHistoryRepo.findLatest(userId);
    if (!lastChange) return;

    const ageHours = (Date.now() - lastChange.createdAt.getTime()) / (1000 * 60 * 60);
    if (ageHours < this.MIN_PASSWORD_AGE_HOURS) {
      throw new ValidationError(
        'PASSWORD_TOO_NEW',
        'Password must be at least 24 hours old before it can be changed again.'
      );
    }
  }

  async saveToHistory(userId: string, passwordHash: string): Promise<void> {
    await this.passwordHistoryRepo.save({ userId, passwordHash, createdAt: new Date() });
    // Keep only last N entries
    await this.passwordHistoryRepo.pruneOld(userId, this.HISTORY_COUNT);
  }
}
```

---


---

## 9. Last Login Reporting & Credential Spray Detection

```typescript
// ✅ Good — show last login info at next successful login
// Store last login info on every successful authentication
async function recordSuccessfulLogin(userId: string, req: Request): Promise<void> {
  const previous = await userRepo.getLastLogin(userId); // fetch BEFORE updating

  await userRepo.updateLastLogin(userId, {
    lastLoginAt:  new Date(),
    lastLoginIp:  req.ip,
    lastLoginAgent: req.headers['user-agent'],
  });

  // Return previous login info to include in the login response
  return previous;
}

// ✅ In login response — inform user of their last session
async function login(dto: LoginDto, req: Request): Promise<LoginResponse> {
  const user   = await validateCredentials(dto);
  const previous = await recordSuccessfulLogin(user.id, req);
  const tokens = await generateTokens(user);

  return {
    ...tokens,
    lastLogin: previous ? {
      at:        previous.lastLoginAt,
      ipAddress: previous.lastLoginIp,   // NOT the full IP in response — just awareness
      message:   `Last login: ${previous.lastLoginAt.toISOString()}`,
    } : null,
  };
}

// ✅ Good — credential spray detection (same password, many accounts)
// This bypasses per-account lockouts by trying one password against many usernames
async function detectCredentialSpray(password: string, ip: string): Promise<void> {
  // Hash the password for key (so we don't store plain password in Redis)
  const passwordKey = crypto.createHash('sha256').update(password + SPRAY_PEPPER).digest('hex');
  const key         = `spray:${ip}:${passwordKey.slice(0, 16)}`; // partial hash — good enough

  const attempts = await redis.incr(key);
  await redis.expire(key, 3600); // 1 hour window

  if (attempts > 10) { // same password tried against >10 different accounts in 1 hour
    logger.warn('Credential spray attack detected', { ip, attemptCount: attempts });
    throw new SecurityError('SPRAY_DETECTED', 'Suspicious login pattern detected. Try again later.');
  }
}
```

---


---

## 10. Temporary Password Enforcement & Vendor Credential Rotation

```typescript
// ✅ Good — force password change on first use of temporary password
interface UserAuthState {
  id:                     string;
  passwordHash:           string;
  isTemporaryPassword:    boolean;  // set to true when admin creates or resets
  temporaryPasswordSetAt: Date | null;
  passwordExpiresAt:      Date | null;
}

// ✅ Middleware: intercept ALL requests if temp password not yet changed
async function enforceTempPasswordChange(req: Request, res: Response, next: NextFunction) {
  const user = req.user as UserAuthState;
  if (!user) return next();

  // Allow only the change-password endpoint if using a temporary password
  if (user.isTemporaryPassword && req.path !== '/auth/change-password') {
    return res.status(403).json({
      error: {
        code:    'TEMPORARY_PASSWORD_CHANGE_REQUIRED',
        message: 'You must change your temporary password before continuing.',
        redirectTo: '/auth/change-password',
      }
    });
  }
  next();
}

// ✅ Good — change all vendor/default credentials at deployment
// In your infrastructure bootstrap script:
async function bootstrapDefaultCredentials(): Promise<void> {
  const defaultAdminExists = await userRepo.findByEmail('admin@localhost');
  if (defaultAdminExists) {
    logger.warn('Default admin account found — disabling');
    await userRepo.disable(defaultAdminExists.id);
  }

  const defaultDbUser = process.env.DB_DEFAULT_USER;
  if (defaultDbUser === 'root' || defaultDbUser === 'admin') {
    throw new Error('Default DB credentials detected. Change before deployment.');
  }
}
```

---


---

