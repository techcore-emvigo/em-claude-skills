# Error Handling & Structured Logging

Covers: domain error classes, async error handling, structured JSON logging,
log levels, request-scoped trace IDs, PII masking, per-environment log levels.

---

# Error Handling & Logging Best Practices

## Core Principle
Every error must be caught, enriched with context, and either handled or propagated with
that context intact. Silent failures are the hardest bugs to diagnose in production.

---

## 1. Error Handling Patterns

### Never Swallow Exceptions

```typescript
// ❌ Critical — error is lost completely
try {
  await paymentService.charge(order);
} catch (e) {}

// ❌ Major — error logged but context lost, execution continues silently
try {
  await paymentService.charge(order);
} catch (e) {
  console.error(e);
}

// ✅ Good — error enriched with context, re-thrown for caller to handle
try {
  await paymentService.charge(order);
} catch (error) {
  logger.error('Payment charge failed', {
    orderId: order.id,
    amount: order.totalAmount,
    error: error.message,
    stack: error.stack,
  });
  throw new PaymentProcessingError(
    `Failed to charge order ${order.id}: ${error.message}`,
    { cause: error }
  );
}
```

### Domain-Specific Error Classes

Create error classes per domain — don't throw generic `Error` everywhere.
This lets callers handle specific error types and lets error filters map to HTTP status codes precisely.

```typescript
// errors/base.error.ts
export class AppError extends Error {
  constructor(
    public readonly code: string,
    message: string,
    public readonly context?: Record<string, unknown>,
    options?: ErrorOptions,
  ) {
    super(message, options);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

// errors/domain.errors.ts
export class UserNotFoundError extends AppError {
  constructor(userId: string) {
    super('USER_NOT_FOUND', `User ${userId} not found`, { userId });
  }
}

export class PaymentProcessingError extends AppError {
  constructor(message: string, options?: ErrorOptions) {
    super('PAYMENT_PROCESSING_FAILED', message, undefined, options);
  }
}

export class ValidationError extends AppError {
  constructor(field: string, message: string) {
    super('VALIDATION_ERROR', message, { field });
  }
}
```

```typescript
// ✅ Usage — controller maps domain errors to HTTP status
@Catch(AppError)
export class AppExceptionFilter implements ExceptionFilter {
  catch(error: AppError, host: ArgumentsHost) {
    const statusMap: Record<string, number> = {
      USER_NOT_FOUND:           404,
      VALIDATION_ERROR:         422,
      UNAUTHORIZED:             401,
      PAYMENT_PROCESSING_FAILED: 502,
    };
    const status = statusMap[error.code] ?? 500;
    const response = host.switchToHttp().getResponse();
    response.status(status).json({
      error: {
        code:      error.code,
        message:   status < 500 ? error.message : 'An unexpected error occurred',
        requestId: response.locals.requestId,
      },
    });
  }
}
```

### Async Error Handling

```typescript
// ❌ Bad — unhandled promise rejection, no error context
function loadUserData(userId: string) {
  fetchUser(userId).then(user => {
    saveToCache(user);
  });
}

// ✅ Good — every async path handled
async function loadUserData(userId: string): Promise<void> {
  try {
    const user = await fetchUser(userId);
    await saveToCache(user);
    logger.info('User data cached', { userId });
  } catch (error) {
    logger.error('Failed to load and cache user data', { userId, error: error.message });
    throw error; // propagate — don't silently fail
  }
}
```

### Try-Catch in Every Service Method

```typescript
// ✅ Pattern: service method with full error lifecycle
class OrderService {
  async createOrder(dto: CreateOrderDto, userId: string): Promise<Order> {
    logger.info('Creating order', { userId, itemCount: dto.items.length });

    try {
      const user = await this.userRepo.findById(userId);
      if (!user) throw new UserNotFoundError(userId);

      const order = Order.create(user, dto.items);
      await this.orderRepo.save(order);

      logger.info('Order created successfully', { orderId: order.id, userId });
      this.eventBus.publish(new OrderCreatedEvent(order));
      return order;

    } catch (error) {
      if (error instanceof AppError) throw error; // re-throw domain errors as-is
      // Wrap unexpected errors with context
      throw new AppError(
        'ORDER_CREATION_FAILED',
        'Unexpected error creating order',
        { userId, itemCount: dto.items.length },
        { cause: error },
      );
    }
  }
}
```

---

## 2. Structured Logging

### Log Format — Always JSON in Production

Every log entry must include these fields. Use a logger library (Winston, Pino, Bunyan) —
never raw `console.log` in production code.

```typescript
// logging/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  base: {
    service:     process.env.SERVICE_NAME,
    environment: process.env.NODE_ENV,
    version:     process.env.APP_VERSION,
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  // In production: output JSON. In development: pretty-print
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty' }
    : undefined,
});
```

### Log Levels — Use the Right Level

| Level | When to Use | Example |
|---|---|---|
| `error` | Unexpected failure, action required | DB connection lost, payment charge failed |
| `warn` | Degraded behaviour, approaching a limit | Retry attempt 2/3, cache miss rate high |
| `info` | Significant lifecycle events | Service started, user registered, order placed |
| `debug` | Developer detail, disabled in production | SQL query executed, cache key checked |

```typescript
// ❌ Bad — wrong levels, no context, no structure
console.log('user created');
console.log('Error: ' + err);
console.error('something went wrong');

// ✅ Good — correct level, structured context
logger.info('User registered', {
  userId:    user.id,
  traceId:   req.traceId,
  plan:      user.plan,
  source:    'registration-api',
});

logger.warn('Payment retry attempt', {
  orderId:    order.id,
  attempt:    attemptNumber,
  maxAttempts: MAX_RETRY_ATTEMPTS,
  traceId:    req.traceId,
});

logger.error('Payment charge failed', {
  orderId:    order.id,
  traceId:    req.traceId,
  error:      error.message,
  stack:      error.stack,
});
```

### Request-Scoped Logging with Trace ID

Propagate a `traceId` / `requestId` through the entire request lifecycle
so all log lines for one request can be correlated in your log aggregator.

```typescript
// middleware/request-context.middleware.ts
import { v4 as uuid } from 'uuid';
import { AsyncLocalStorage } from 'async_hooks';

export const requestContext = new AsyncLocalStorage<{ traceId: string }>();

export function requestContextMiddleware(req, res, next) {
  const traceId = req.headers['x-trace-id'] ?? uuid();
  res.setHeader('x-trace-id', traceId);
  requestContext.run({ traceId }, next);
}

// In logger: always include traceId from context
export function getLogger() {
  const ctx = requestContext.getStore();
  return logger.child({ traceId: ctx?.traceId ?? 'no-context' });
}
```

### What NEVER to Log (PII / Secrets Masking)

```typescript
// ❌ Critical — PII and credentials in logs
logger.info('User login', {
  email:    user.email,      // PII — never log
  password: dto.password,    // credential — never log
  phone:    user.phone,      // PII — never log
  cardNumber: card.number,   // financial PII — never log
});

// ❌ Critical — full API response logged (may contain tokens)
logger.debug('Stripe response', { response: stripeFullResponse });

// ✅ Good — log identifiers and metadata only, mask sensitive fields
logger.info('User login', {
  userId:  user.id,                           // ID only, not email
  plan:    user.plan,
  traceId: req.traceId,
});

logger.info('Card payment processed', {
  userId:         user.id,
  lastFour:       card.number.slice(-4),      // only last 4 digits
  paymentIntentId: stripeResponse.id,         // external reference ID only
  traceId:        req.traceId,
});
```

### Masking Utility

```typescript
// utils/mask.utils.ts
export const mask = {
  email:  (email: string)  => email.replace(/(.{2})(.*)(@.*)/, '$1***$3'),
  phone:  (phone: string)  => phone.slice(0, 3) + '****' + phone.slice(-2),
  card:   (card: string)   => '**** **** **** ' + card.slice(-4),
  token:  (token: string)  => token.slice(0, 6) + '...',
};

// Usage
logger.warn('Login failed', {
  maskedEmail: mask.email(dto.email),  // 'jo***@example.com'
  traceId:     req.traceId,
});
```

---

## 3. Logging in Every Layer

Log at meaningful boundaries — not every line, but at every significant transition:

```typescript
// ✅ Controller — log incoming request (no body — may contain PII)
logger.info('POST /orders received', { userId, traceId });

// ✅ Service — log business event outcomes
logger.info('Order created', { orderId: order.id, userId, traceId });

// ✅ Repository — log only in debug (too verbose for info)
logger.debug('Query executed', { table: 'orders', durationMs, traceId });

// ✅ Third-party integration — log call metadata (not full response)
logger.info('Stripe charge initiated', { paymentIntentId, amount, traceId });
logger.info('Stripe charge succeeded', { paymentIntentId, traceId });

// ✅ Queue consumer — log processing lifecycle
logger.info('Processing event', { eventId, eventType, traceId });
logger.info('Event processed', { eventId, eventType, durationMs, traceId });
```

---

## 4. Log Level Management Per Environment

```typescript
// config/logger.config.ts
const LOG_LEVELS = {
  production:  'warn',   // only warn + error in prod — info is too verbose
  staging:     'info',   // full info in staging for debugging
  development: 'debug',  // everything in local dev
  test:        'error',  // silence logs during tests
} as const;

export const LOG_LEVEL = LOG_LEVELS[process.env.NODE_ENV ?? 'development'];
```

---

## Gap Detection Table

| Gap | What to Look For | Severity |
|---|---|---|
| Swallowed exception | `catch (e) {}` or `catch (e) { return null; }` | 🔴 |
| `console.log` as logger | `console.log`, `console.error` in production code | 🟡 |
| No structured logging | Log messages as plain strings, not JSON objects | 🟠 |
| PII in log statement | `email`, `password`, `phone`, `ssn` in any log call | 🔴 |
| Missing `traceId` in logs | Log entries with no request correlation field | 🟠 |
| Generic `Error` thrown | `throw new Error('failed')` with no code or context | 🟡 |
| Error caught and not rethrown | `catch (e) { logger.error(e) }` — execution silently continues | 🟠 |
| Wrong log level | `error` used for expected validation failures | 🟡 |
| No log on important events | Order placed, payment processed, user registered — no log entry | 🟠 |
| Missing error context | `logger.error(error.message)` — no orderId, userId, or traceId | 🟠 |
| Stack trace in API response | `res.json({ error: error.stack })` — internal detail exposed | 🟠 |
| No domain-specific error classes | Only `Error` used — callers can't distinguish error types | 🟡 |

---

