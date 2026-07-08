# General Coding Practices — Part 2: Integrity & Safe Deployment

Covers: integrity verification (checksums/hashes), least privilege, managed code
over custom implementations, dependency review, secure auto-update, process hardening.

---

## 9. Numeric Safety — Precision, Overflow, Coercion

```typescript
// ❌ Bad — precision loss with large integers
const largeId = 9007199254740993;    // > MAX_SAFE_INTEGER: silent precision loss
// ✅ Good
const largeId = BigInt('9007199254740993');

// ❌ Bad — float arithmetic on money
const total = 0.1 + 0.2;            // 0.30000000000000004
const cents = 19.99 * 100;          // 1998.9999999999998
// ✅ Good — work in integer cents
const cents       = Math.round(19.99 * 100);   // 1999
// ✅ Good — decimal library for financial calculations
import Decimal from 'decimal.js';
const total = new Decimal('19.99').times(3).toFixed(2); // '59.97'

// ❌ Bad — NaN propagates silently
const qty      = parseInt(req.query.qty);  // NaN if missing
const subtotal = qty * price;             // NaN — silent order corruption
// ✅ Good — validate every numeric conversion
const qty = Number(req.query.qty);
if (!Number.isFinite(qty) || qty <= 0 || !Number.isInteger(qty)) {
  throw new ValidationError('qty', 'Must be a positive integer');
}

// ❌ Bad — silent truncation via bitwise
const truncated = 300 & 0xFF;            // 44 — silently loses upper bits
// ✅ Good — validate range before any narrowing operation
function toByte(value: number, field: string): number {
  if (!Number.isInteger(value) || value < 0 || value > 255) {
    throw new ValidationError(field, 'Must be an integer 0-255');
  }
  return value;
}

// ❌ Bad — type coercion surprise
'5' + 3;         // '53' — string concat, not addition!
// ✅ Good — explicit conversion with radix
const a = Number(req.body.x);
const b = parseInt(req.body.y, 10); // always supply radix 10
```

---

## 10. Integrity Verification — Checksums & Hashes

Verify integrity of all downloaded files, config files, and libraries before use.

```typescript
// ✅ Good — SHA-256 integrity check before using any downloaded content
async function downloadAndVerify(
  url:            string,
  expectedSha256: string,
  destPath:       string,
): Promise<void> {
  if (!url.startsWith('https://')) {
    throw new SecurityError('Downloads must use HTTPS');
  }

  const fileBuffer = await fetchBuffer(url);
  const actualHash = crypto.createHash('sha256').update(fileBuffer).digest('hex');

  if (!crypto.timingSafeEqual(
    Buffer.from(actualHash.toLowerCase()),
    Buffer.from(expectedSha256.toLowerCase()),
  )) {
    throw new SecurityError(
      `Integrity check failed. Expected: ${expectedSha256}. Got: ${actualHash}`
    );
  }

  await fs.writeFile(destPath, fileBuffer);
  logger.info('File downloaded and integrity verified', { sha256: actualHash });
}

// ✅ Good — verify config file integrity at startup to detect tampering
async function loadVerifiedConfig(configPath: string, expectedHash: string): Promise<Config> {
  const raw  = await fs.readFile(configPath, 'utf-8');
  const hash = crypto.createHash('sha256').update(raw).digest('hex');

  if (hash !== expectedHash) {
    logger.error('Config file tampered', { path: configPath, expected: expectedHash, actual: hash });
    throw new SecurityError('Config file integrity check failed');
  }

  return JSON.parse(raw) as Config;
}
```

---

## 11. Least Privilege — Raise and Drop Elevated Permissions ASAP

```typescript
// ❌ Bad — admin credentials held for entire process lifetime
const adminClient = createAdminClient(ADMIN_KEY);

// ✅ Good — acquire just-in-time, release immediately after one operation
async function performPrivilegedOperation(resourceId: string): Promise<void> {
  const adminClient = await createAdminClient();
  try {
    await adminClient.grantAccess(resourceId);
  } finally {
    await adminClient.revoke();
    adminClient.destroy();
  }
}

// ✅ Good — AWS STS: minimum-duration scoped credentials
async function getTemporaryUploadCredentials(bucket: string): Promise<AwsCredentials> {
  const response = await sts.send(new AssumeRoleCommand({
    RoleArn:         process.env.S3_WRITE_ROLE_ARN,
    RoleSessionName: `upload-${Date.now()}`,
    DurationSeconds: 900, // 15 minutes minimum
    Policy: JSON.stringify({
      Statement: [{
        Effect: 'Allow', Action: ['s3:PutObject'],
        Resource: `arn:aws:s3:::${bucket}/uploads/*`, // narrowest scope
      }],
    }),
  }));
  return response.Credentials;
}
```

---

## 12. Managed Code — Use Vetted Libraries

Never re-implement cryptography, parsing, authentication, or date arithmetic.
Use well-tested, maintained libraries for all security-sensitive operations.

```typescript
// ❌ Bad — custom crypto and token generation
function myHash(s: string): string { /* custom — not secure */ }
const token = Math.random().toString(36);  // predictable!

// ✅ Good — use Node.js built-ins and vetted libraries
import crypto from 'crypto';
import argon2 from 'argon2';
import bcrypt from 'bcrypt';

const hash  = await argon2.hash(password);
const token = crypto.randomBytes(32).toString('hex'); // secure
const uuid  = crypto.randomUUID();                    // Node 14.17+
const b64   = Buffer.from(data).toString('base64');   // built-in

// ❌ Bad — custom XML/HTML parser
const value = xml.match(/<tag>(.*?)<\/tag>/)?.[1]; // fragile regex

// ✅ Good — proper parser
import { XMLParser }  from 'fast-xml-parser';
import DOMPurify      from 'isomorphic-dompurify';

const parsed   = new XMLParser().parse(xmlString);
const safeHtml = DOMPurify.sanitize(userHtml, { ALLOWED_TAGS: ['b', 'i', 'p'] });

// ❌ Bad — manual date arithmetic (DST, leap year bugs)
const expires = Date.now() + 7 * 24 * 60 * 60 * 1000;

// ✅ Good — date-fns handles edge cases
import { addDays } from 'date-fns';
const expires = addDays(new Date(), 7);
```

---

## 13. Third-Party Code & Dependency Review

Every dependency is potential attack surface. Review before adding; audit continuously.

```typescript
/*
  Pre-add dependency checklist (enforce in PR template):
  [ ] Does a Node.js built-in solve this? (fetch, crypto.randomUUID, structuredClone)
  [ ] npm audit: zero high/critical CVEs on the package
  [ ] Last published: not abandoned (>2 years without update = review carefully)
  [ ] Weekly downloads: reasonable adoption (not a 3-download typosquat)
  [ ] Source reviewed: exact package name verified (not a lookalike)
  [ ] Licence: compatible (no GPL-2.0 in commercial products)
  [ ] Transitive deps: not pulling in 200 packages for one function
*/
```

```json
{
  "dependencies": {
    "jsonwebtoken": "9.0.2",
    "bcrypt":       "5.1.1",
    "express":      "4.19.2",
    "axios":        "1.7.2"
  },
  "scripts": {
    "audit":      "npm audit --audit-level=high",
    "check-deps": "npx depcheck",
    "licenses":   "npx license-checker --failOn 'GPL-2.0;AGPL-3.0'"
  },
  "engines": { "node": ">=20.0.0" }
}
```

```yaml
# .github/workflows/security.yml
name: Security Audit
on: [push, pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: CVE scan
        run: npm audit --audit-level=high
      - name: Snyk scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
      - name: Licence check
        run: npx license-checker --failOn 'GPL-2.0;AGPL-3.0'
```

---

## 14. Secure Auto-Update — Signatures and Encrypted Channels

```typescript
// ✅ Good — full secure update pipeline
class SecureUpdater {

  async fetchAndVerify(version: string): Promise<Buffer> {
    const baseUrl = process.env.UPDATE_SERVER_URL;
    if (!baseUrl.startsWith('https://')) {
      throw new SecurityError('Update server must use HTTPS — never HTTP');
    }

    const [pkg, sig] = await Promise.all([
      this.httpsClient.getBuffer(`${baseUrl}/${version}.tar.gz`),
      this.httpsClient.getBuffer(`${baseUrl}/${version}.tar.gz.sig`),
    ]);

    // Verify RSA-PSS signature with pinned public key (bundled — not fetched)
    const publicKey = await this.loadPinnedPublicKey();
    const isValid   = crypto.verify(
      'sha256', pkg,
      { key: publicKey, padding: crypto.constants.RSA_PKCS1_PSS_PADDING },
      sig,
    );
    if (!isValid) throw new SecurityError('Update signature invalid — possible tampering');

    // SHA-256 checksum against published manifest (second factor)
    const actualHash   = crypto.createHash('sha256').update(pkg).digest('hex');
    const manifestHash = await this.fetchManifestHash(version);
    if (!crypto.timingSafeEqual(Buffer.from(actualHash), Buffer.from(manifestHash))) {
      throw new SecurityError('Update checksum mismatch');
    }

    logger.info('Update verified', { version, sha256: actualHash });
    return pkg;
  }
}
```

---

## 15. Non-Executable Memory & Process Hardening

For Node.js/web applications, the equivalent of non-executable stack protection:

```typescript
// ✅ Node.js startup flags
// node --disallow-code-generation-from-strings server.js
// Prevents eval(), new Function(), vm.runInContext() at runtime

// ✅ Freeze prototypes in production to prevent prototype pollution
if (process.env.NODE_ENV === 'production') {
  Object.freeze(Object.prototype);
  Object.freeze(Function.prototype);
  Object.freeze(Array.prototype);
}

// ✅ CSP: block inline scripts in browser (browser-equivalent of NX)
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc:  ["'self'"],  // no inline scripts, no eval
    objectSrc:  ["'none'"],
    frameAncestors: ["'none'"],
  },
}));
```

```yaml
# Kubernetes securityContext — OS-level memory protection
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem:   true   # code pages unwritable
  runAsNonRoot:             true
  runAsUser:                10001
  capabilities:
    drop: ["ALL"]
# Writable scratch space via tmpfs only
volumes:
  - name: tmp
    emptyDir:
      medium: Memory
```

---

## Gap Detection Table

### Resource Management

| Gap | What to Look For | Severity |
|---|---|---|
| DB connection not released in finally | `conn.query()` with no `finally { conn.release() }` | 🔴 |
| File stream not closed on exception | `createReadStream()` with no `.destroy()` in finally | 🟠 |
| Timer not cleared | `setInterval(...)` with no `clearInterval` on all exit paths | 🟠 |
| Event listener not removed | `addEventListener` without cleanup on unmount/destroy | 🟠 |
| In-memory store without eviction | Module-level `Map`/array growing with no TTL or size cap | 🟠 |

### Buffer & Memory Safety

| Gap | What to Look For | Severity |
|---|---|---|
| Fixed buffer in loop with no boundary check | `buf.copy(out, offset)` in loop without overflow guard | 🔴 |
| `Buffer.allocUnsafe()` for user-facing data | Uninitialized memory sent to users | 🟠 |
| No bounds check before buffer read | `buf.readUInt32BE(offset)` without verifying `offset + 4 <= buf.length` | 🟠 |
| Null bytes not trimmed from fixed-width field | `buf.toString()` without trimming at first `\x00` | 🟡 |
| No aggregate size cap before allocation | Chunks merged without total size validation | 🟠 |
| Unbounded user string to string operation | User input concatenated without truncation | 🟠 |

### Dangerous Functions

| Gap | What to Look For | Severity |
|---|---|---|
| `eval(nonLiteral)` | `eval(variable)`, `eval(req.body.x)` | 🔴 |
| `new Function(nonLiteral)` | Dynamic function construction from user string | 🔴 |
| `setTimeout(string, ...)` | String (not function) as first argument | 🔴 |
| OS `exec`/`system` with user input | Shell command with request data interpolated | 🔴 |
| `pickle.loads(userBytes)` Python | Untrusted bytes deserialized with pickle | 🔴 |
| `subprocess.call(cmd, shell=True)` | shell=True with any dynamic input | 🔴 |
| `strcpy`/`strcat`/`sprintf` in C code | Unbounded string operation with external data | 🔴 |
| `printf(userInput)` without format string | User input as format argument | 🔴 |

### Numeric Safety

| Gap | What to Look For | Severity |
|---|---|---|
| `parseInt()` result not validated | Used without `isNaN`/`isFinite` check | 🟠 |
| Float arithmetic on money values | `price * qty` without rounding | 🟠 |
| `Number` for large integer IDs | IDs > `MAX_SAFE_INTEGER` losing precision | 🟠 |
| NaN propagated silently | `NaN * price` in business logic without guard | 🟠 |
| Silent bitwise truncation | `value & 0xFF` without range validation | 🟠 |

### Concurrency

| Gap | What to Look For | Severity |
|---|---|---|
| TOCTOU: check then act without lock | `findById()` + `update()` without atomic DB op | 🔴 |
| No distributed lock on shared resource | Multiple processes can act on same item | 🔴 |
| Non-atomic counter | `count = count + 1` instead of `INCR` / Redis `INCR` | 🟠 |
| Shared state mutated without protection | Module-level variable written by async handlers | 🟠 |

### Managed Code & Dependencies

| Gap | What to Look For | Severity |
|---|---|---|
| Custom hash/crypto implementation | Hand-rolled hash, PRNG, or token generation | 🔴 |
| `Math.random()` for security token | Token/session/OTP from `Math.random()` | 🔴 |
| Custom JWT parsing without signature | Manual base64 + JSON.parse, no signature check | 🔴 |
| Shell command instead of library | `exec('convert ...')`, `exec('rm ...')` in app code | 🟠 |
| `package-lock.json` deleted or missing | No lockfile — no integrity hashes | 🟠 |
| No `npm audit` in CI pipeline | Dependency CVE scan not in build | 🔴 |
| Security-critical dep with `^`/`~` version | JWT/auth/framework versions not pinned | 🟠 |
| Third-party lib added without review | No security/licence check before merge | 🟠 |

### Integrity & Auto-Update

| Gap | What to Look For | Severity |
|---|---|---|
| Downloaded file used without hash check | File executed without SHA-256 verification | 🔴 |
| Auto-update without signature check | Package applied without RSA/ECDSA verification | 🔴 |
| Update downloaded over HTTP | Auto-update URL using `http://` | 🔴 |
| Public signing key fetched remotely | Key downloaded from server (can be replaced) | 🔴 |
| Config file not integrity-checked | Config loaded without hash verification | 🟠 |
