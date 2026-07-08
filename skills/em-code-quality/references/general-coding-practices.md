# General Coding Practices — Part 1

Covers: variable initialisation, no dynamic execution of user data,
race conditions and locking, numeric safety and overflow prevention.

---

# General Coding Practices

Covers: variable initialisation, no dynamic execution of user data, race conditions,
numeric safety, integrity verification, least privilege, managed code, dependency
review, secure auto-update, non-executable memory hardening.

---

## 6. Explicit Variable Initialisation

Always initialise at declaration or immediately before first use.
Uninitialised variables cause silent bugs, undefined behaviour, and
security issues when garbage memory contains sensitive data from prior use.

```typescript
// ❌ Bad — result undefined if condition is false
let result;
let errorCode;

if (condition) {
  result    = processData();
  errorCode = 0;
}
processResult(result); // undefined if condition was false

// ✅ Good — initialise at declaration with typed defaults
let result:    ProcessResult | null = null;
let errorCode: number               = -1;

// ✅ Better — const removes the problem entirely
const result    = condition ? processData() : getDefaultResult();
const errorCode = condition ? 0 : ERROR_CODES.NOT_PROCESSED;

// ✅ Good — every struct field explicitly set
const summary: OrderSummary = {
  orderId:   id,
  total:     0,        // not undefined
  itemCount: 0,        // not undefined
  hasErrors: false,    // not undefined
};
```

```python
# ❌ Bad — total None causes TypeError on first active item
total = None
for item in items:
    if item.active:
        total = total + item.price  # TypeError!

# ✅ Good — explicit zero initialisation
total: float = 0.0
for item in items:
    if item.active:
        total += item.price
```

---

## 7. No Dynamic Execution of User-Supplied Data

User input must NEVER reach any function that evaluates or executes code or shell commands.
Use a whitelist dispatch map instead of `eval`, `exec`, `require(variable)`, or shell strings.

```typescript
// ❌ Critical — user controls code path
eval(req.body.formula);
new Function(req.body.code)();
require(`./handlers/${req.query.action}`);      // loads any module
exec(`convert ${req.query.file} output.jpg`);  // OS command injection

// ✅ Good — whitelist dispatch: user picks a key, server picks the function
const FORMULA_OPS: Record<string, (a: number, b: number) => number> = {
  add:      (a, b) => a + b,
  subtract: (a, b) => a - b,
  multiply: (a, b) => a * b,
  discount: (price, pct) => price * (1 - pct / 100),
};

function evaluateFormula(op: string, a: number, b: number): number {
  const fn = FORMULA_OPS[op];
  if (!fn) throw new ValidationError('op', `Unknown operation: ${op}`);
  return fn(a, b); // our function — not the user's code
}

// ✅ Good — OS tasks via dedicated library APIs, never shell strings
import sharp   from 'sharp';
import ffmpeg  from 'fluent-ffmpeg';

async function resizeImage(buf: Buffer, w: number, h: number): Promise<Buffer> {
  return sharp(buf).resize(w, h, { fit: 'inside' }).webp({ quality: 85 }).toBuffer();
}

async function convertVideo(safeInputPath: string, safeOutputPath: string): Promise<void> {
  return new Promise((resolve, reject) =>
    ffmpeg(safeInputPath)
      .output(safeOutputPath)
      .on('end', resolve)
      .on('error', reject)
      .run()
  );
}

// ✅ Good — safe file path from user param (prevent traversal)
function resolveUserPath(userParam: string, baseDir: string): string {
  const resolved = path.resolve(baseDir, path.basename(userParam));
  if (!resolved.startsWith(path.resolve(baseDir))) {
    throw new SecurityError('Path traversal attempt blocked');
  }
  return resolved;
}
```

---

## 8. Race Conditions & Concurrent Access

Shared resources accessed concurrently must be protected with atomic DB operations,
advisory locks, or distributed locks. Read-then-write without a lock is always a race.

```typescript
// ❌ Bad — TOCTOU: two requests both see 'available', both reserve the same seat
async function reserveSeat(seatId: string, userId: string): Promise<boolean> {
  const seat = await seatRepo.findById(seatId);
  if (seat.status === 'available') {
    await seatRepo.update(seatId, { status: 'reserved', userId });
    return true;
  }
  return false;
}

// ✅ Good — single atomic conditional UPDATE at the DB level
async function reserveSeat(seatId: string, userId: string): Promise<boolean> {
  const result = await db.query(
    `UPDATE seats
     SET    status = 'reserved', user_id = $1, updated_at = NOW()
     WHERE  id = $2 AND status = 'available'
     RETURNING id`,
    [userId, seatId]
  );
  return result.rowCount > 0; // true = got it; false = already taken
}

// ✅ Good — distributed lock for cross-service critical sections
async function withLock<T>(key: string, ttlSecs: number, fn: () => Promise<T>): Promise<T> {
  const acquired = await redis.set(`lock:${key}`, '1', 'NX', 'EX', ttlSecs);
  if (!acquired) throw new ConflictError('Resource locked by another process');
  try {
    return await fn();
  } finally {
    await redis.del(`lock:${key}`); // always release
  }
}

// ✅ Good — atomic counter with Redis INCR (never read-increment-write)
async function checkRateLimit(userId: string, maxPerHour: number): Promise<void> {
  const key   = `rl:${userId}:${getHourBucket()}`;
  const count = await redis.incr(key);           // atomic
  if (count === 1) await redis.expire(key, 3600);
  if (count > maxPerHour) throw new TooManyRequestsError();
}
```

---

