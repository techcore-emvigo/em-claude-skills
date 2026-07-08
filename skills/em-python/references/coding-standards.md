# Coding Standards & Naming Conventions

Covers: naming rules (camelCase, PascalCase, snake_case per language), function/class
size limits, constants, enums, comments, formatting, DRY, immutability, linting.

---

# Coding Standards & Naming Conventions

## Why This Matters
Well-named, consistently formatted code reduces onboarding time, prevents bugs from
misunderstanding, and makes automated tooling (linting, search, refactoring) reliable.
These rules apply across every language in the stack.

---

## 1. Naming Conventions

### Universal Rules
- Names must be **intention-revealing** — a new developer should understand purpose without reading the implementation
- No abbreviations unless universally known (`url`, `id`, `api`, `http`)
- No single-letter names except loop counters (`i`, `j`) and well-known math variables
- No misleading names — `userList` must be a list, `getUser` must return a user

### Per-Language Conventions

| Construct | JavaScript / TypeScript | Python | C# (.NET) |
|---|---|---|---|
| Variable | `camelCase` | `snake_case` | `camelCase` |
| Function / Method | `camelCase` | `snake_case` | `PascalCase` |
| Class | `PascalCase` | `PascalCase` | `PascalCase` |
| Constant | `UPPER_SNAKE_CASE` | `UPPER_SNAKE_CASE` | `PascalCase` |
| Interface | `IUserService` (TS) | N/A | `IUserService` |
| Enum | `PascalCase` (values: `UPPER_SNAKE`) | `PascalCase` (values: `UPPER_SNAKE`) | `PascalCase` |
| File (JS/TS) | `kebab-case.ts` | `snake_case.py` | `PascalCase.cs` |
| React Component | `PascalCase` | N/A | N/A |
| CSS Class | `kebab-case` | N/A | N/A |
| DB Table | `snake_case` (plural) | | |
| DB Column | `snake_case` | | |

### ❌ Bad vs ✅ Good Naming Examples

```typescript
// ❌ Bad — abbreviations, no intent, wrong case
const usrLst = await db.find();
function proc(d: any) { return d.filter((x: any) => x.a === 1); }
class data_manager {}
const MAX = 86400000;

// ✅ Good — intention-revealing, consistent
const activeUsers = await userRepository.findAllActive();
function filterVerifiedOrders(orders: Order[]): Order[] {
  return orders.filter(order => order.status === OrderStatus.VERIFIED);
}
class UserSessionManager {}
const SESSION_TIMEOUT_MS = 86_400_000; // 24 hours — numeric separator for readability
```

```typescript
// ❌ Bad — boolean trap (what do true/false mean?)
createUser(userData, true, false, true);

// ✅ Good — named options object
createUser(userData, {
  sendWelcomeEmail: true,
  requireEmailVerification: false,
  isAdminCreated: true,
});
```

---

## 2. Function & Class Size Limits

| Rule | Limit | Rationale |
|---|---|---|
| Function body | ≤ 30 lines | Each function does one thing |
| Class / file | ≤ 200–300 lines | Single responsibility |
| Function parameters | ≤ 4 | Use parameter object beyond 4 |
| Nesting depth | ≤ 3 levels | Use early returns to flatten |
| Cyclomatic complexity | ≤ 10 (SonarQube default) | Simplify branching logic |

```typescript
// ❌ Bad — deep nesting, does too many things
function processOrder(order: any) {
  if (order) {
    if (order.items) {
      if (order.items.length > 0) {
        for (const item of order.items) {
          if (item.stock > 0) {
            // 20 more lines of logic...
          }
        }
      }
    }
  }
}

// ✅ Good — early returns, single responsibility, named helpers
function processOrder(order: Order): void {
  if (!order?.items?.length) return;
  const availableItems = order.items.filter(hasStock);
  availableItems.forEach(reserveItem);
}

const hasStock = (item: OrderItem): boolean => item.stockQuantity > 0;
const reserveItem = (item: OrderItem): void => inventoryService.reserve(item.sku);
```

---

## 3. Constants & Magic Values

Never use unexplained literals. Every constant needs a name that explains its meaning.

```typescript
// ❌ Bad — magic numbers and strings scattered through code
if (user.role === 2) { ... }
setTimeout(refresh, 300000);
if (password.length < 8) { ... }
const url = 'https://api.stripe.com/v1/charges';

// ✅ Good — named constants grouped by domain
// constants/user.constants.ts
export const UserRole = {
  ADMIN: 2,
  MEMBER: 1,
  GUEST: 0,
} as const;

// constants/auth.constants.ts
export const AUTH = {
  MIN_PASSWORD_LENGTH: 8,
  TOKEN_REFRESH_INTERVAL_MS: 5 * 60 * 1000, // 5 minutes
  SESSION_TIMEOUT_MS: 60 * 60 * 1000,        // 1 hour
} as const;

// constants/payment.constants.ts
export const STRIPE_API = {
  BASE_URL: process.env.STRIPE_BASE_URL,     // from env, not hardcoded
  CHARGES_PATH: '/v1/charges',
} as const;
```

---

## 4. Enumerations (Enums)

Use enums for any group of related constants — status values, types, error codes, roles.
Never use raw strings or magic numbers for finite sets of values.

```typescript
// ❌ Bad — magic strings scattered across codebase
if (order.status === 'pending') { ... }
if (order.status === 'PENDING') { ... }  // inconsistent even within the same project

// ✅ Good — enum enforces valid values at compile time
enum OrderStatus {
  PENDING   = 'PENDING',
  CONFIRMED = 'CONFIRMED',
  SHIPPED   = 'SHIPPED',
  DELIVERED = 'DELIVERED',
  CANCELLED = 'CANCELLED',
}

if (order.status === OrderStatus.PENDING) { ... }
```

```typescript
// ✅ Good — error code enum for consistent API responses
enum ErrorCode {
  VALIDATION_FAILED   = 'VALIDATION_FAILED',
  UNAUTHORIZED        = 'UNAUTHORIZED',
  RESOURCE_NOT_FOUND  = 'RESOURCE_NOT_FOUND',
  PAYMENT_FAILED      = 'PAYMENT_FAILED',
  INTERNAL_ERROR      = 'INTERNAL_ERROR',
}
```

---

## 5. Comments & Documentation

### When to Comment
- **Always**: non-obvious business rules, performance trade-offs, known limitations
- **Never**: restating what the code obviously does

```typescript
// ❌ Bad — comments that restate the code
// Loop through users
for (const user of users) {
  // Check if user is active
  if (user.isActive) {
    // Send email
    sendEmail(user);
  }
}

// ✅ Good — comment explains WHY, not WHAT
// Trial users receive a reminder 3 days before expiry.
// Paid users do not receive reminders — they manage billing via the portal.
const eligibleUsers = users.filter(u => u.isActive && u.plan === Plan.TRIAL);
await notificationService.sendExpiryReminders(eligibleUsers);
```

### File / Module Header
Every file should have a brief header explaining its purpose:
```typescript
/**
 * UserAuthService
 *
 * Handles all user authentication flows: login, registration,
 * token refresh, and logout. JWT signing and validation is
 * delegated to JwtService.
 *
 * @module auth
 */
```

### Comment Formatting Rules
- Place comments on their own line — never trailing after code
- Begin with capital letter, end with period
- One space after `//`
- Use `// TODO:` for known issues (resolve before release), `// FIXME:` for bugs

```typescript
// ❌ Bad
const x = calculate(); // calculate value

// ✅ Good
// Compound interest calculated daily to match bank statement reconciliation.
const accruedInterest = calculateDailyCompoundInterest(principal, rate, days);

// TODO: Replace with streaming query once table exceeds 1M rows.
const results = await db.query('SELECT * FROM events');
```

---

## 6. Code Formatting Rules

These are enforced by ESLint / Prettier / SonarLint — they should fail CI if violated.

| Rule | Standard |
|---|---|
| Indent | 2 spaces (JS/TS), 4 spaces (Python/C#) |
| Max line length | 100–120 characters |
| Trailing commas | Always (multiline) |
| Semicolons | Consistent — always or never, never mixed |
| Quotes | Single quotes (JS/TS) or configured in Prettier |
| Blank lines | One blank line between methods; two before class |
| Trailing whitespace | Never |
| Final newline | Always |

```typescript
// ❌ Bad — inconsistent, long lines, mixed style
const result = someReallyLongFunctionName(firstArgument,secondArgument,thirdArgument,fourthArgument,fifthArgument)
var x= 'hello'
const y = "world"

// ✅ Good — consistent, readable, Prettier-compliant
const result = someReallyLongFunctionName(
  firstArgument,
  secondArgument,
  thirdArgument,
  fourthArgument,
  fifthArgument,
);
const greeting = 'hello';
const subject = 'world';
```

---

## 7. Avoiding Duplication (DRY)

Every piece of knowledge should have a single authoritative representation.
Duplication means two places to update when logic changes — which means one always gets missed.

```typescript
// ❌ Bad — same validation logic duplicated in 3 endpoints
// In createUser.controller.ts
if (!email.includes('@') || email.length < 5) throw new Error('Invalid email');

// In updateProfile.controller.ts
if (!email.includes('@') || email.length < 5) throw new Error('Invalid email');

// ✅ Good — single validation function, used everywhere
// validators/email.validator.ts
export function validateEmail(email: string): void {
  if (!EMAIL_REGEX.test(email)) {
    throw new ValidationError(ErrorCode.INVALID_EMAIL, `Invalid email: ${email}`);
  }
}
```

---

## 8. Immutability

Prefer immutable data. Never mutate function arguments. Use `const` by default.

```typescript
// ❌ Bad — mutates the input array
function addDiscount(items: OrderItem[], discount: number): OrderItem[] {
  for (let i = 0; i < items.length; i++) {
    items[i].price *= (1 - discount); // mutates caller's data!
  }
  return items;
}

// ✅ Good — returns a new array, input unchanged
function applyDiscount(items: OrderItem[], discountRate: number): OrderItem[] {
  return items.map(item => ({
    ...item,
    price: item.price * (1 - discountRate),
    discountApplied: discountRate,
  }));
}
```

---

## 9. Tooling — SonarLint & Linting

Install and configure these locally and in CI:

| Tool | Purpose | Config File |
|---|---|---|
| **SonarLint** (IDE plugin) | Real-time code quality and security feedback | Connected to SonarQube rules |
| **ESLint** | JS/TS lint rules enforcement | `.eslintrc.json` |
| **Prettier** | Code formatting | `.prettierrc` |
| **Google Style Guide** | JS style reference | https://google.github.io/styleguide/jsguide.html |
| **Pylint / Ruff** | Python lint | `pyproject.toml` |
| **SonarQube** | CI gate for quality, security, coverage | `sonar-project.properties` |

**Minimum ESLint rules to enforce:**
```json
{
  "rules": {
    "no-var": "error",
    "prefer-const": "error",
    "eqeqeq": ["error", "always"],
    "no-console": "warn",
    "no-unused-vars": "error",
    "no-duplicate-imports": "error",
    "max-lines-per-function": ["warn", { "max": 30 }],
    "complexity": ["warn", 10],
    "no-magic-numbers": ["warn", { "ignore": [0, 1, -1] }]
  }
}
```

---

## Gap Detection Table

| Gap | What to Look For | Severity |
|---|---|---|
| Typos in identifiers | `getUsersById`, `fecthOrder`, `caluculate` | 🟡 |
| Inconsistent naming convention | Mix of `camelCase` and `snake_case` in same file | 🟡 |
| Magic numbers/strings | Unexplained numeric or string literals in logic | 🟡 |
| Raw strings for enum values | `if (status === 'active')` instead of `OrderStatus.ACTIVE` | 🟡 |
| Function > 30 lines | Single function with mixed responsibilities | 🟠 |
| Class > 200 lines | God class with multiple concerns | 🟠 |
| Parameters > 4 | `fn(a, b, c, d, e)` with no options object | 🟡 |
| No file/module header | File has no description of purpose | 🟡 |
| Code-restating comments | Comments that just say what the next line does | 🟡 |
| TODO left unresolved | `// TODO:` in code heading to production | 🟠 |
| Commented-out code | `// const old = doOldThing()` in production codebase | 🟡 |
| Trailing whitespace | Whitespace at end of lines | 🟡 |
| No blank line between methods | Dense code with no visual separation | 🟡 |
| Inconsistent semicolons | Some lines end with `;`, some don't in same file | 🟡 |
| Duplicated logic blocks | Same 5+ line logic appears in 2+ places | 🟠 |

---

