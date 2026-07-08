# JavaScript — Advanced Practices

Covers: strict mode, modern ECMAScript, this binding rules, LogDNA structured
logging, standardised API response service, Sentry integration.

---

## 5. Strict Mode & Modern ECMAScript

```javascript
// ✅ Good — 'use strict' in non-module files (CommonJS / legacy scripts)
// NOTE: ES modules (type: "module" or .mjs) are always strict — don't add manually
'use strict';

// ❌ Bad — calling 'use strict' inside a function or in production bundled output
function init() {
  'use strict'; // too narrow scope — apply at file level
}
```

**ECMAScript best practices:**
```javascript
// ✅ Use optional chaining (ES2020)
const city = user?.address?.city ?? 'Unknown';

// ✅ Use nullish coalescing — not || (falsy trap)
const timeout = config.timeout ?? 5000;  // 0 is a valid timeout, || would override it

// ✅ Use logical assignment operators (ES2021)
user.role  ??= 'member';     // assign only if null/undefined
config.debug ||= false;      // assign only if falsy

// ✅ Use Array methods effectively
const orderIds   = orders.map(o => o.id);
const pending    = orders.filter(o => o.status === OrderStatus.PENDING);
const total      = orders.reduce((sum, o) => sum + o.amount, 0);
const hasOverdue = orders.some(o => o.dueDate < new Date());
const allPaid    = orders.every(o => o.isPaid);

// ✅ Use spread / rest operators
const updatedUser = { ...user, lastLoginAt: new Date() };  // immutable update
function log(level, message, ...meta) { /* rest params */ }
```

---

## 6. `this` Binding Rules

```javascript
// ❌ Bad — assigning `this` to a variable
function Timer() {
  const self = this;          // anti-pattern
  setTimeout(function() {
    self.tick();
  }, 1000);
}

// ✅ Good — use arrow function (captures `this` lexically)
function Timer() {
  setTimeout(() => {
    this.tick();              // arrow function: `this` from outer scope
  }, 1000);
}

// ✅ Good — use .bind() when passing method as callback
class EventEmitter {
  constructor() {
    this.handleClick = this.handleClick.bind(this);  // bind once in constructor
  }
  handleClick(event) { /* `this` is correct */ }
}

// ✅ Good — use .call() / .apply() for explicit `this` context
function greet() { return `Hello, ${this.name}`; }
greet.call({ name: 'Alice' });   // 'Hello, Alice'
greet.apply({ name: 'Bob' });    // 'Hello, Bob'
```

---

## 7. Logging — LogDNA / Structured Logger (Not console.log)

```javascript
// ❌ Bad — console.log pollutes production logs, not searchable, no levels
console.log('user created:', user);
console.log('error:', err);

// ✅ Good — use a structured logger (Winston, Pino, or LogDNA SDK)
import logger from './logger';  // centralised logger module

// logger.ts — single logger instance for the whole service
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  base:  { service: process.env.SERVICE_NAME, env: process.env.NODE_ENV },
  timestamp: pino.stdTimeFunctions.isoTime,
});

export default logger;

// ✅ Usage — structured, searchable, levelled
logger.info({ userId: user.id, action: 'user_created' }, 'User registered.');
logger.warn({ attempt, maxAttempts, orderId }, 'Payment retry attempt.');
logger.error({ err, orderId, userId }, 'Order creation failed.');
// LogDNA, Datadog, ELK, CloudWatch can parse and index these JSON logs
```

---

## 8. Standardised API Response Service

All API responses must be formatted by a single response helper.
Never build response objects inline per endpoint — ensures consistency across all APIs.

```typescript
// response/api-response.ts — single source of truth for all response shapes
export class ApiResponse<T> {

  static success<T>(data: T, meta?: Record<string, unknown>) {
    return { success: true, data, ...(meta && { meta }) };
  }

  static paginated<T>(data: T[], total: number, page: number, limit: number) {
    return {
      success: true,
      data,
      meta: { total, page, limit, totalPages: Math.ceil(total / limit) },
    };
  }

  static error(code: string, message: string, requestId: string, details?: unknown) {
    return {
      success: false,
      error: { code, message, requestId, ...(details && { details }) },
    };
  }
}

// ✅ Usage in controllers — consistent shape everywhere
@Get('orders')
async listOrders(@Query() query: ListOrdersQuery) {
  const { data, total } = await this.orderService.list(query);
  return ApiResponse.paginated(data, total, query.page, query.limit);
}

@Post('orders')
async createOrder(@Body() dto: CreateOrderDto) {
  const order = await this.orderService.create(dto);
  return ApiResponse.success(order);
}
```

---

## 9. Sentry Integration (All Frontend & Mobile)

Every frontend (web and mobile) must integrate Sentry error monitoring.

```typescript
// ✅ Good — Sentry initialisation (React / Next.js)
// sentry.client.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn:         process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release:     process.env.NEXT_PUBLIC_APP_VERSION,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  // Never send PII to Sentry
  beforeSend(event) {
    // Strip sensitive fields before sending to Sentry
    if (event.user) {
      delete event.user.email;
      delete event.user.ip_address;
    }
    return event;
  },
});
```

```typescript
// ✅ Good — Sentry in React Native
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn:         process.env.SENTRY_DSN,
  environment: process.env.APP_ENV,
  release:     `${APP_ID}@${APP_VERSION}+${BUILD_NUMBER}`,
  tracesSampleRate: 1.0,
});

// Wrap the root component
export default Sentry.wrap(App);
```

```typescript
// ✅ Good — capture errors with context, not raw
try {
  await paymentService.charge(order);
} catch (error) {
  Sentry.withScope(scope => {
    scope.setTag('feature', 'payment');
    scope.setContext('order', { orderId: order.id, amount: order.total });
    // DO NOT add user.email or card details to scope
    Sentry.captureException(error);
  });
  throw error;
}
```

---

## Gap Detection Table (Additions)

| Gap | What to Look For | Severity |
|---|---|---|
| `'use strict'` inside function | `function fn() { 'use strict'; }` | 🟡 |
| `const self = this` | Variable assigned `this` instead of arrow function or `.bind()` | 🟡 |
| `console.log` used as logger | `console.log(`, `console.info(` in service/business logic | 🟡 |
| No Sentry integration in frontend | No `@sentry/react`, `@sentry/nextjs`, or `@sentry/react-native` | 🔴 |
| Inline API response objects | `res.json({ data: ..., total: ..., page: ... })` per endpoint | 🟡 |
| No response service / helper | Every endpoint builds its own response shape differently | 🟠 |
| `||` used for default where `0` is valid | `config.timeout || 5000` when `0` is a valid timeout | 🟡 |
| Callback inside async function | `asyncFn(function(err, data) { ... })` mixed with async/await | 🟠 |
| No package-lock.json | Missing lock file or deleted from repo | 🟠 |
| Node version not specified | No `engines` in `package.json` and no `.nvmrc` | 🟡 |
