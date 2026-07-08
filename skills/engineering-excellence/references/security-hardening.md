# Security Hardening — Core Controls

Covers: secrets management, input validation layers, auth/authz checks,
security headers (helmet/CSP), dependency auditing, CAPTCHA, rate limiting,
no PII in cookies.

---

# Security Hardening — Configuration & Defence

Covers: secrets management, input validation, security headers, dependency auditing,
CAPTCHA, rate limiting, PII in cookies, CDN SRI, OWASP Top 10, cache-control,
one-time tokens, cookie/header PII prevention.

---

# Security Best Practices — Code Reference

## 1. Secrets & Configuration Management

Never store secrets in code. Secrets belong in environment variables or a secrets manager.

```typescript
// ❌ Critical — hardcoded secrets in source code
const stripeClient = new Stripe('sk_live_real_key_here');
const dbConfig = { password: 'P@ssword123' };
const jwtSecret = 'my-secret-key';

// ❌ Critical — secrets in a committed config file
// config.json  (committed to git)
{ "stripe_key": "sk_live_abc123", "db_password": "realpass" }

// ✅ Good — secrets from environment / secrets manager
// .env (in .gitignore)  |  AWS SSM  |  Vault  |  Firebase Remote Config
const stripeClient = new Stripe(process.env.STRIPE_SECRET_KEY);
const jwtSecret    = process.env.JWT_SECRET;

// ✅ Good — validated config at startup (fail fast if misconfigured)
// config/app.config.ts
import { z } from 'zod';

const configSchema = z.object({
  STRIPE_SECRET_KEY: z.string().min(1),
  JWT_SECRET:        z.string().min(32),
  DB_PASSWORD:       z.string().min(1),
  NODE_ENV:          z.enum(['development', 'staging', 'production', 'test']),
});

export const config = configSchema.parse(process.env); // throws if any env var missing
```

---

## 2. Input Validation — Client, API, and Persistence Layer

Validate at every boundary. Client-side validation is UX only — always validate on the server.

```typescript
// ❌ Bad — raw request body used directly
app.post('/users', async (req, res) => {
  await db.users.create(req.body); // mass assignment, no validation
});

// ✅ Good — Zod schema validates and transforms at the boundary
import { z } from 'zod';

const CreateUserSchema = z.object({
  name:     z.string().min(2).max(100).trim(),
  email:    z.string().email().toLowerCase(),
  password: z.string().min(8).max(128),
  role:     z.enum(['member', 'admin']).default('member'),
});

app.post('/users', async (req, res) => {
  const result = CreateUserSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({
      error: { code: 'VALIDATION_ERROR', issues: result.error.flatten() }
    });
  }
  const dto = result.data; // type-safe, validated, sanitized
  await userService.create(dto);
});
```

### Validation at Every Layer

```typescript
// Layer 1: API boundary (above example — Zod/class-validator)

// Layer 2: Service / domain layer — business rule validation
class Order {
  static create(items: OrderItem[]): Order {
    if (!items.length) throw new ValidationError('items', 'Order must have at least one item');
    if (items.length > 50) throw new ValidationError('items', 'Order cannot exceed 50 items');
    return new Order(items);
  }
}

// Layer 3: DB / persistence layer — DB constraints as last line of defence
// In migration:
// ALTER TABLE orders ADD CONSTRAINT chk_items_count CHECK (items_count > 0);
```

---

## 3. Authentication & Authorization

```typescript
// ❌ Bad — trusting role from request body
app.post('/admin/users', (req, res) => {
  if (req.body.role === 'admin') { // user controls their own role!
    createAdminUser(req.body);
  }
});

// ✅ Good — role from verified JWT payload only
// auth/jwt.strategy.ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  validate(payload: JwtPayload) {
    // payload is verified by JWT signature — tamper-proof
    return { userId: payload.sub, role: payload.role };
  }
}

// ✅ Good — route protected by guard + role check
@Post('admin/users')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.ADMIN)
async createAdminUser(@Body() dto: CreateUserDto, @CurrentUser() actor: User) {
  logger.info('Admin user creation', { actorId: actor.id, targetEmail: mask.email(dto.email) });
  return this.userService.createAdmin(dto);
}
```

### Row-Level Authorization (Ownership Check)

```typescript
// ❌ Bad — any authenticated user can read any order
@Get('orders/:id')
@UseGuards(JwtAuthGuard)
async getOrder(@Param('id') id: string) {
  return this.orderRepo.findById(id); // no ownership check!
}

// ✅ Good — verify ownership before returning data
@Get('orders/:id')
@UseGuards(JwtAuthGuard)
async getOrder(@Param('id') id: string, @CurrentUser() user: AuthUser) {
  const order = await this.orderRepo.findById(id);
  if (!order) throw new UserNotFoundError(id);
  if (order.userId !== user.id && user.role !== UserRole.ADMIN) {
    throw new ForbiddenException('Access denied');
  }
  return new OrderResponseDto(order);
}
```

### JWT — Expiry and Token Storage

```typescript
// ✅ Good — short-lived access token, HttpOnly cookie for refresh
const ACCESS_TOKEN_TTL  = '1h';    // expires in 1 hour
const REFRESH_TOKEN_TTL = '7d';    // refresh token lives 7 days

// Access token: in-memory only (never localStorage — XSS risk)
// Refresh token: HttpOnly, Secure, SameSite=Strict cookie
res.cookie('refreshToken', refreshToken, {
  httpOnly: true,
  secure:   process.env.NODE_ENV === 'production',
  sameSite: 'strict',
  maxAge:   7 * 24 * 60 * 60 * 1000, // 7 days in ms
});
```

---

## 4. Security Headers

```typescript
// ✅ Good — helmet sets all critical security headers in one call
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc:  ["'self'"],
      scriptSrc:   ["'self'"],          // no inline scripts, no external scripts
      styleSrc:    ["'self'", "'unsafe-inline'"],
      imgSrc:      ["'self'", 'data:', 'https://cdn.yourdomain.com'],
      connectSrc:  ["'self'", 'https://api.yourdomain.com'],
      frameAncestors: ["'none'"],       // X-Frame-Options: DENY equivalent
    },
  },
  xFrameOptions:         { action: 'deny' },          // clickjacking protection
  xContentTypeOptions:   true,                         // no MIME sniffing
  strictTransportSecurity: {
    maxAge: 31536000,
    includeSubDomains: true,
  },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
}));

// ✅ Good — CORS restricted to known origins
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') ?? [],
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
}));
// ❌ Never: cors({ origin: '*' }) with credentials: true
```

---

## 5. Dependency & Library Security

```typescript
// package.json — pin major versions, audit regularly
{
  "scripts": {
    "audit":        "npm audit --audit-level=high",
    "audit:fix":    "npm audit fix",
    "check-updates": "npx npm-check-updates"
  }
}
```

**Rules:**
- Run `npm audit` in CI — fail on high/critical CVEs
- Review and update dependencies every 6 months minimum
- Remove unused dependencies: `npx depcheck`
- Never use direct JS/CSS/font CDN URLs from third-party providers in production HTML — host them yourself or use a Subresource Integrity (SRI) hash

```html
<!-- ❌ Bad — third-party CDN failure takes down your app -->
<script src="https://cdn.somelib.com/library.js"></script>

<!-- ✅ Good — self-hosted or SRI hash locks the resource -->
<script
  src="https://cdn.somelib.com/library.min.js"
  integrity="sha384-abc123def456..."
  crossorigin="anonymous">
</script>
```

---

## 6. CAPTCHA on Public APIs & Forms

```typescript
// ✅ Good — reCAPTCHA v3 on registration and contact forms
// Frontend: obtain token
const token = await grecaptcha.execute('SITE_KEY', { action: 'register' });

// Backend: verify token before processing
async function verifyCaptcha(token: string): Promise<boolean> {
  const response = await fetch('https://www.google.com/recaptcha/api/siteverify', {
    method: 'POST',
    body:   new URLSearchParams({
      secret:   process.env.RECAPTCHA_SECRET_KEY,
      response: token,
    }),
  });
  const data = await response.json();
  return data.success && data.score >= 0.5; // score 0.0 (bot) to 1.0 (human)
}

app.post('/register', async (req, res) => {
  const captchaValid = await verifyCaptcha(req.body.captchaToken);
  if (!captchaValid) return res.status(400).json({ error: 'Captcha verification failed' });
  // proceed with registration
});
```

---

## 7. Rate Limiting

```typescript
// ✅ Good — rate limiting on all public and auth endpoints
import rateLimit from 'express-rate-limit';

// Strict limit for auth endpoints
const authLimiter = rateLimit({
  windowMs:    15 * 60 * 1000, // 15 minutes
  max:         10,              // 10 attempts per window
  message:     { error: { code: 'RATE_LIMITED', message: 'Too many attempts' } },
  standardHeaders: true,
  legacyHeaders:   false,
});

// General API limit
const apiLimiter = rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max:      100,
});

app.use('/auth',   authLimiter);
app.use('/api',    apiLimiter);
```

---

## 8. No PII in Cookies

```typescript
// ❌ Bad — PII in cookie
res.cookie('user', JSON.stringify({ email: user.email, name: user.name }));

// ✅ Good — only opaque session ID in cookie
res.cookie('sessionId', session.id, {
  httpOnly: true,
  secure:   true,
  sameSite: 'strict',
});
// Server looks up session data by ID — no PII ever leaves the server in a cookie
```

---

## Gap Detection Table

| Gap | What to Look For | Severity |
|---|---|---|
| Hardcoded secret | String literal matching key/secret/password/token pattern | 🔴 |
| No input validation on API | `req.body` or `req.query` used without schema validation | 🟠 |
| Role from request body | `if (req.body.role === 'admin')` | 🔴 |
| No ownership check | Resource fetched by ID with no `userId` comparison | 🔴 |
| Token in localStorage | `localStorage.setItem('token', ...)` | 🔴 |
| No CAPTCHA on public form | Registration/contact/password-reset with no CAPTCHA | 🟠 |
| CORS wildcard | `cors({ origin: '*' })` on authenticated API | 🔴 |
| Missing security headers | No `helmet()` middleware | 🟠 |
| CSP not set | No `Content-Security-Policy` header | 🟠 |
| External JS/CSS CDN with no SRI | Third-party CDN URLs without integrity hash | 🟠 |
| No rate limiting on auth endpoint | Login/register with no throttle | 🔴 |
| JWT expiry > 1 hour | `expiresIn: '24h'` on access token | 🟠 |
| PII in cookie | `email`, `name`, `phone` serialized into cookie value | 🔴 |
| Vulnerable dependency | `npm audit` reports high/critical CVE | 🔴 |
| Dependency not updated in 6+ months | Outdated packages with known vulnerabilities | 🟠 |

---

