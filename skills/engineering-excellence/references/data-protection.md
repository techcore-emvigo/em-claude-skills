# Data Protection

Covers: encryption at rest, no sensitive data in GET params, autocomplete
disable, cache-control on sensitive pages, comment removal, data retention.

---

## 1. Encrypt Sensitive Data at Rest

Never store authentication data, PII, or secrets in plain text — even server-side.

```typescript
// ✅ Good — AES-256-GCM encryption service for sensitive fields
// services/encryption.service.ts
import crypto from 'crypto';

const ALGORITHM  = 'aes-256-gcm';
const KEY_LENGTH = 32;   // 256 bits
const IV_LENGTH  = 12;   // 96 bits for GCM

@Injectable()
export class EncryptionService {

  private readonly key: Buffer;

  constructor() {
    const keyHex = process.env.FIELD_ENCRYPTION_KEY;
    if (!keyHex || keyHex.length !== KEY_LENGTH * 2) {
      throw new Error('FIELD_ENCRYPTION_KEY must be a 64-character hex string');
    }
    this.key = Buffer.from(keyHex, 'hex');
  }

  encrypt(plaintext: string): string {
    const iv         = crypto.randomBytes(IV_LENGTH); // unique per encryption
    const cipher     = crypto.createCipheriv(ALGORITHM, this.key, iv);
    const encrypted  = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
    const authTag    = cipher.getAuthTag(); // GCM authentication tag

    // Format: iv(hex):tag(hex):ciphertext(hex)
    return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted.toString('hex')}`;
  }

  decrypt(encryptedValue: string): string {
    const [ivHex, tagHex, ciphertextHex] = encryptedValue.split(':');
    const iv         = Buffer.from(ivHex, 'hex');
    const authTag    = Buffer.from(tagHex, 'hex');
    const ciphertext = Buffer.from(ciphertextHex, 'hex');

    const decipher = crypto.createDecipheriv(ALGORITHM, this.key, iv);
    decipher.setAuthTag(authTag); // GCM verifies integrity — throws if tampered
    return decipher.update(ciphertext) + decipher.final('utf8');
  }
}

// ✅ Usage — encrypt PII fields before storing
// In user entity / repository
async saveUserWithEncryption(user: CreateUserDto): Promise<void> {
  await this.db.users.create({
    id:              generateId(),
    email:           user.email.toLowerCase(),  // email is used as lookup — store plain
    phoneEncrypted:  this.encryption.encrypt(user.phone),   // encrypted at rest
    ssnEncrypted:    this.encryption.encrypt(user.ssn),     // encrypted at rest
    dobEncrypted:    this.encryption.encrypt(user.dob),     // encrypted at rest
    passwordHash:    await argon2.hash(user.password),      // hashed
  });
}
```

---

## 2. Sensitive Data in HTTP GET Parameters

```typescript
// ❌ Critical — sensitive data in GET URL (logged in server/proxy/browser history)
GET /users/search?email=john@example.com&ssn=123-45-6789
GET /auth/verify?token=super-secret-token&password=abc123

// ✅ Good — sensitive data only via POST body (encrypted in transit via HTTPS)
@Post('auth/verify')
async verifyEmail(@Body() dto: VerifyEmailDto): Promise<void> {
  // token in POST body — not in URL, not in server logs
  await this.authService.verifyEmailToken(dto.token);
}

// ✅ Good — search by sensitive fields only server-side
@Post('admin/users/search')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.ADMIN)
async searchUsers(@Body() dto: UserSearchDto): Promise<UserResponse[]> {
  // Sensitive search criteria in POST body, not GET query params
  return this.userService.search(dto);
}

// ✅ Good — filter referer header when linking to external sites
// If a page URL contains sensitive params, strip them before external redirect
function buildExternalLink(externalUrl: string, pageUrl: string): string {
  // Don't let external sites see your internal URL via Referer header
  return `<a href="${externalUrl}" rel="noreferrer noopener">link</a>`;
  // rel="noreferrer" — browser won't send Referer header
}
```

---

## 3. Disable Autocomplete on Sensitive Forms

```html
<!-- ❌ Bad — browser autocomplete fills in sensitive fields -->
<form action="/login" method="POST">
  <input type="text"     name="email"    />
  <input type="password" name="password" />
</form>

<!-- ✅ Good — autocomplete disabled on auth forms; off on sensitive data fields -->
<form action="/login" method="POST" autocomplete="off">
  <input type="email"    name="email"    autocomplete="username"         />
  <input type="password" name="password" autocomplete="current-password" />
</form>

<!-- For non-login sensitive fields -->
<input type="text"  name="ssn"         autocomplete="off" />
<input type="text"  name="cardNumber"  autocomplete="off" />
<input type="text"  name="cvv"         autocomplete="off" />
```

---

## 4. Cache-Control on Sensitive Pages

```typescript
// ✅ Good — prevent browser/proxy caching of authenticated and sensitive pages
// middleware/no-cache.middleware.ts
export function noCacheMiddleware(req: Request, res: Response, next: NextFunction) {
  res.setHeader('Cache-Control', 'no-cache, no-store, must-revalidate, private');
  res.setHeader('Pragma',        'no-cache');     // HTTP/1.0 backward compat
  res.setHeader('Expires',       '0');
  next();
}

// Apply to all authenticated routes
app.use('/api', noCacheMiddleware);
app.use('/dashboard', noCacheMiddleware);
app.use('/account', noCacheMiddleware);

// ✅ Good — allow caching only on truly public, non-sensitive content
app.get('/public/products', (req, res) => {
  res.setHeader('Cache-Control', 'public, max-age=300, stale-while-revalidate=60');
  res.json(products);
});
```

---

## 5. Remove Source Code Comments Before Production

```typescript
// ❌ Bad — comments reveal internal architecture details
function processPayment(amount: number) {
  // TODO: Fix the race condition in the DB on line 47 of payment.repo.ts
  // HACK: bypass Stripe verification for test accounts (accountId starts with 'test_')
  // DB table: payments_v2, column: stripe_charge_id — changed in migration_044
  // Admin backdoor: if (amount === 0) skip auth check
  return chargeCard(amount);
}

// ✅ Good — production code has no implementation-revealing comments
// Track todos in your issue tracker, not in source code comments
function processPayment(amount: number): Promise<PaymentResult> {
  // Payment amounts are in minor currency units (cents)
  return this.stripeService.charge(amount);
}
```

---

## 6. Data Retention & Removal

```typescript
// ✅ Good — support removal of sensitive data on request (GDPR Right to Erasure)
// services/data-retention.service.ts
@Injectable()
export class DataRetentionService {

  // GDPR right to erasure — anonymise rather than hard delete
  async eraseUserData(userId: string, requestedBy: string): Promise<void> {
    logger.info('Data erasure initiated', { userId, requestedBy });

    await this.db.$transaction(async (tx) => {
      // 1. Anonymise user record
      await tx.user.update({
        where: { id: userId },
        data: {
          email:         `erased_${userId}@deleted.local`,  // anonymised
          name:          'Deleted User',
          phoneEncrypted: null,
          ssnEncrypted:   null,
          dobEncrypted:   null,
          erasedAt:       new Date(),
          erasedBy:       requestedBy,
        },
      });

      // 2. Anonymise personal data in related records
      await tx.order.updateMany({
        where:  { userId },
        data:   { shippingAddressEncrypted: null },
      });

      // 3. Revoke all sessions
      await this.sessionStore.revokeAll(userId);
    });

    await this.auditService.log({
      actorId:    requestedBy,
      action:     'user.data_erased',
      resourceId: userId,
    });
  }

  // Automated purge of temporary sensitive files
  @Cron('0 3 * * *') // daily at 3 AM
  async purgeExpiredTempFiles(): Promise<void> {
    const expiredFiles = await this.tempFileRepo.findExpired();
    for (const file of expiredFiles) {
      await this.storageService.delete(file.path);
      await this.tempFileRepo.delete(file.id);
    }
    logger.info('Temp files purged', { count: expiredFiles.length });
  }
}
```

---

