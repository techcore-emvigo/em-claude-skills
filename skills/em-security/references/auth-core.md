# Authentication & Password Management — Core

Covers: password hashing (bcrypt/argon2), policy enforcement, account lockout,
vague error messages, password reset flow, re-authentication, MFA.

---

# Authentication & Password Management

Covers: password hashing, lockout, vague error messages, password policy, MFA,
reset flow, re-authentication, vendor credential rotation, external auth.

---

## 3. Authentication & Password Management

### Password Hashing — Server-Side Only

```typescript
// ❌ Critical — weak hashing
const hash = md5(password);       // broken — reversible with rainbow tables
const hash = sha256(password);    // better but still brute-forceable without salt

// ✅ Good — bcrypt with salt (cost factor 12 minimum)
import bcrypt from 'bcrypt';

const BCRYPT_ROUNDS = 12;   // minimum; use 14 for high-security contexts

async function hashPassword(plaintext: string): Promise<string> {
  return bcrypt.hash(plaintext, BCRYPT_ROUNDS);  // bcrypt generates a unique salt per hash
}

async function verifyPassword(plaintext: string, hash: string): Promise<boolean> {
  return bcrypt.compare(plaintext, hash);  // timing-safe comparison built in
}

// ✅ Good — argon2 (stronger than bcrypt, recommended for new systems)
import argon2 from 'argon2';

async function hashPassword(plaintext: string): Promise<string> {
  return argon2.hash(plaintext, {
    type:        argon2.argon2id,  // argon2id is the recommended variant
    memoryCost:  65536,            // 64MB memory cost
    timeCost:    3,                // 3 iterations
    parallelism: 4,
  });
}
```

### Password Policy Enforcement

```typescript
// ✅ Good — centralised password policy
const PasswordSchema = z.string()
  .min(8,   'Password must be at least 8 characters (16+ recommended)')
  .max(128, 'Password must be at most 128 characters')
  .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
  .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
  .regex(/[0-9]/, 'Password must contain at least one number')
  .regex(/[^A-Za-z0-9]/, 'Password must contain at least one special character');

// ✅ Good — password field obscured in HTML (never type="text")
// <input type="password" name="password" autocomplete="current-password" />

// ✅ Good — "disable remember me" for password fields
// <input type="password" autocomplete="off" />   // for sensitive forms
```

### Vague Authentication Error Messages

```typescript
// ❌ Bad — reveals which field is wrong (username enumeration attack)
if (!user) {
  return res.status(401).json({ error: 'Invalid username' });
}
if (!await verifyPassword(dto.password, user.passwordHash)) {
  return res.status(401).json({ error: 'Invalid password' });
}

// ✅ Good — identical message regardless of which field failed
// Never reveal whether the username or password was wrong
const user = await userRepo.findByEmail(dto.email);
const passwordValid = user ? await verifyPassword(dto.password, user.passwordHash) : false;

if (!user || !passwordValid) {
  // Same message, same HTTP status, same response time (no timing oracle)
  return res.status(401).json({
    error: { code: 'AUTH_FAILED', message: 'Invalid email and/or password' }
  });
}
```

### Account Lockout After Failed Attempts

```typescript
// ✅ Good — lockout after N failed attempts, with exponential delay
const MAX_ATTEMPTS = 5;
const LOCKOUT_DURATION_MS = 15 * 60 * 1000; // 15 minutes

async function trackFailedLogin(email: string): Promise<void> {
  const key     = `login:failures:${email}`;
  const attempts = await redis.incr(key);
  await redis.expire(key, 900); // 15-minute window

  if (attempts >= MAX_ATTEMPTS) {
    const lockKey = `login:locked:${email}`;
    await redis.set(lockKey, '1', 'EX', LOCKOUT_DURATION_MS / 1000);
    logger.warn('Account locked after failed attempts', { maskedEmail: mask.email(email) });
  }
}

async function isAccountLocked(email: string): Promise<boolean> {
  return !!(await redis.exists(`login:locked:${email}`));
}

// ✅ Check before processing login
async function login(dto: LoginDto): Promise<AuthTokens> {
  if (await isAccountLocked(dto.email)) {
    throw new AccountLockedError('Account temporarily locked. Try again later.');
  }

  const user = await userRepo.findByEmail(dto.email);
  const valid = user && await verifyPassword(dto.password, user.passwordHash);

  if (!valid) {
    await trackFailedLogin(dto.email);
    throw new AuthenticationError('Invalid email and/or password');  // always same message
  }

  await redis.del(`login:failures:${dto.email}`); // clear on success
  return generateTokens(user);
}
```

### Password Reset — Secure Flow

```typescript
// ✅ Good — complete secure password reset flow
class PasswordResetService {

  async initiateReset(email: string): Promise<void> {
    const user = await userRepo.findByEmail(email);

    // ✅ Always return same response whether user exists or not (prevent email enumeration)
    if (!user) {
      logger.info('Password reset requested for unknown email');
      return; // silently exit — don't reveal account existence
    }

    const token     = crypto.randomBytes(32).toString('hex');
    const expiresAt = new Date(Date.now() + 15 * 60 * 1000); // 15 minutes

    // Store hashed token — never store raw reset tokens
    await passwordResetRepo.save({
      userId:    user.id,
      tokenHash: await bcrypt.hash(token, 10),
      expiresAt,
      used:      false,
    });

    await mailer.sendPasswordReset(user.email, token); // send plain token to user
    logger.info('Password reset email sent', { userId: user.id });
  }

  async completeReset(token: string, newPassword: string): Promise<void> {
    // Find all non-expired, unused resets
    const resets = await passwordResetRepo.findActive();
    const match  = await findMatchingReset(token, resets); // compare bcrypt hashes

    if (!match || match.expiresAt < new Date()) {
      throw new ValidationError('INVALID_TOKEN', 'Reset token is invalid or expired');
    }

    // ✅ Enforce password policy on new password
    PasswordSchema.parse(newPassword);

    // ✅ Prevent password reuse (check last N passwords)
    await enforcePasswordHistory(match.userId, newPassword);

    const newHash = await hashPassword(newPassword);
    await userRepo.updatePassword(match.userId, newHash);

    // ✅ Invalidate token after single use
    await passwordResetRepo.markUsed(match.id);

    // ✅ Notify user of password change
    const user = await userRepo.findById(match.userId);
    await mailer.sendPasswordChangeConfirmation(user.email);

    // ✅ Invalidate all existing sessions on password change
    await sessionStore.revokeAll(match.userId);

    logger.info('Password reset completed', { userId: match.userId });
  }
}
```

### Re-authentication for Critical Operations

```typescript
// ✅ Good — require password confirmation before critical operations
@Delete('account')
@UseGuards(JwtAuthGuard)
async deleteAccount(
  @CurrentUser() user: AuthUser,
  @Body() dto: ConfirmPasswordDto,
): Promise<void> {
  // Re-authenticate before destructive action
  const isValid = await verifyPassword(dto.password, user.passwordHash);
  if (!isValid) {
    throw new AuthenticationError('Password confirmation failed');
  }

  logger.info('Account deletion initiated', { userId: user.id });
  await this.userService.scheduleAccountDeletion(user.id);
}
```

---


---

## 7. Multi-Factor Authentication (MFA)

```typescript
// ✅ Good — TOTP-based MFA for admin and sensitive accounts
import { authenticator } from 'otplib';
import qrcode from 'qrcode';

class MfaService {

  // Generate MFA setup
  async setupMfa(userId: string, email: string): Promise<{ qrCodeUrl: string; secret: string }> {
    const secret  = authenticator.generateSecret(); // 20 bytes of random base32
    const otpauth = authenticator.keyuri(email, process.env.APP_NAME, secret);
    const qrCodeUrl = await qrcode.toDataURL(otpauth);

    // Store encrypted secret — never plain text
    await userRepo.saveMfaSecret(userId, encrypt(secret, process.env.MFA_ENCRYPTION_KEY));
    return { qrCodeUrl, secret };
  }

  // Verify TOTP during login
  async verifyTotp(userId: string, token: string): Promise<boolean> {
    const encryptedSecret = await userRepo.getMfaSecret(userId);
    const secret          = decrypt(encryptedSecret, process.env.MFA_ENCRYPTION_KEY);

    // ✅ TOTP has 30-second windows; allow ±1 window for clock drift
    return authenticator.verify({ token, secret });
  }
}

// ✅ MFA enforced for all admin users
@Post('auth/login')
async login(@Body() dto: LoginDto): Promise<LoginResponse> {
  const user   = await this.authService.validateCredentials(dto);
  const tokens = await this.authService.generateTokens(user);

  if (user.requiresMfa) {
    // Return partial token — forces MFA step before full access
    return { requiresMfa: true, mfaToken: tokens.mfaStepToken };
  }
  return tokens;
}
```

---


---

