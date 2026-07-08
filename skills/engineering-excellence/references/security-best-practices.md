# Authentication Best Practices — Code Reference

Covers: centralised auth architecture, authentication failure defaults,
complete secure registration and login flow with all guards in place.

---

## 14. Centralised Authentication Architecture

```typescript
// ❌ Bad — auth logic scattered: each handler checks its own way
app.get('/orders',   (req, res) => { if (!req.headers.authorization) return res.status(401)... });
app.post('/orders',  (req, res) => { const token = req.cookies.token; if (!token)... });
app.delete('/users', (req, res) => { /* forgot auth check! */ });

// ✅ Good — single centralised auth middleware applied at router level
// auth/auth.middleware.ts — ONE authoritative implementation
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const req   = context.switchToHttp().getRequest();
    const token = this.extractToken(req);

    if (!token) throw new UnauthorizedException('Authentication required');

    try {
      const payload = await this.jwtService.verifyAsync(token, {
        secret:   process.env.JWT_SECRET,
        audience: process.env.JWT_AUDIENCE,  // validate audience claim
        issuer:   process.env.JWT_ISSUER,    // validate issuer claim
      });
      req.user = payload;
    } catch {
      throw new UnauthorizedException('Invalid or expired token');
    }
    return true;
  }

  private extractToken(req: Request): string | null {
    // Accept ONLY from Authorization header — never from URL params
    const [type, token] = req.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : null;
  }
}

// ✅ Applied at controller level — all routes protected by default
@Controller('orders')
@UseGuards(JwtAuthGuard)   // applies to ALL routes in this controller
export class OrdersController {

  @Public()  // explicit opt-out for specific public routes
  @Get('featured')
  getFeatured() { /* public — no auth */ }

  @Get()
  listOrders(@CurrentUser() user: AuthUser) { /* requires auth */ }
}

// ✅ Public decorator marks intentionally open endpoints
export const Public = () => SetMetadata('isPublic', true);
```

---

## 15. Authentication Failure — Secure Defaults

```typescript
// ✅ Good — auth controls FAIL SECURE (deny access on any error)
async function verifyToken(token: string): Promise<JwtPayload> {
  try {
    return await jwtService.verifyAsync(token);
  } catch (error) {
    // On ANY error (expired, malformed, wrong signature, network) — DENY
    // Never "fail open" and allow access when verification fails
    throw new UnauthorizedException('Authentication failed');
  }
}

// ✅ Good — admin functions require AT LEAST as much security as user functions
// Admin endpoints require: JWT (same as users) + MFA + admin role + IP restriction
@Delete('admin/users/:id')
@UseGuards(JwtAuthGuard, MfaVerifiedGuard, RolesGuard, AdminIpWhitelistGuard)
@Roles(UserRole.SUPER_ADMIN)
async deleteUser(@Param('id') id: string) { ... }

// ✅ Segregate auth logic from resources — use middleware/guards, not inline checks
// WRONG: auth check inside business logic
async function getOrder(id: string, authHeader: string) {
  const token = authHeader.split(' ')[1]; // auth mixed into business logic!
  if (!token) throw new Error('Unauthorized');
  return orderRepo.findById(id);
}

// RIGHT: auth handled by guard at framework level, service is auth-unaware
@Get('orders/:id')
@UseGuards(JwtAuthGuard)
async getOrder(@Param('id') id: string, @CurrentUser() user: AuthUser) {
  return this.orderService.findById(id, user.id); // service just does its job
}
```

---

## 16. Complete Secure Registration & Login Flow

```typescript
// ✅ Good — complete production-ready auth service

@Injectable()
export class AuthService {

  async register(dto: RegisterDto): Promise<void> {
    // 1. Validate password policy
    PasswordSchema.parse(dto.password);

    // 2. Check email not already registered (but give vague error)
    const exists = await this.userRepo.existsByEmail(dto.email);
    if (exists) {
      // Do NOT reveal that this email is registered — account enumeration
      // Send "if this email is registered, you will receive a confirmation" pattern
      await this.mailer.sendEmailAlreadyRegistered(dto.email);
      return; // silently succeed from the caller's perspective
    }

    // 3. Hash password server-side
    const hash = await argon2.hash(dto.password, { type: argon2.argon2id, memoryCost: 65536 });

    // 4. Create user — role from system default, NEVER from input
    const user = await this.userRepo.create({
      email:        dto.email.toLowerCase(),
      passwordHash: hash,
      role:         UserRole.MEMBER,  // always default role — never from dto.role
      isActive:     false,            // inactive until email verified
    });

    // 5. Send verification email
    const token = crypto.randomBytes(32).toString('hex');
    await this.emailVerificationRepo.save({
      userId:    user.id,
      tokenHash: await bcrypt.hash(token, 10),
      expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24 hours
    });
    await this.mailer.sendEmailVerification(user.email, token);
    logger.info('User registered, verification email sent', { userId: user.id });
  }

  async login(dto: LoginDto, req: Request): Promise<LoginResponse> {
    // 1. Check credential spray before touching DB
    await this.detectSpray(req.ip, dto.password);

    // 2. Check account lockout
    if (await this.isLocked(dto.email)) {
      throw new AccountLockedError('Account temporarily locked. Try again in 15 minutes.');
    }

    // 3. Fetch user and verify — identical timing regardless of outcome
    const user  = await this.userRepo.findByEmail(dto.email.toLowerCase());
    const valid = user
      ? await argon2.verify(user.passwordHash, dto.password)
      : await argon2.verify(DUMMY_HASH, dto.password); // constant-time even for unknown email

    if (!valid || !user) {
      if (user) await this.recordFailedAttempt(dto.email);
      throw new AuthenticationError('Invalid email and/or password'); // same message always
    }

    if (!user.isActive) {
      throw new AuthenticationError('Account is not active. Check your email for verification.');
    }

    // 4. Clear failed attempts on success
    await redis.del(`login:failures:${dto.email}`);

    // 5. MFA check if enabled
    if (user.mfaEnabled) {
      const mfaToken = await this.issueMfaStepToken(user.id);
      return { requiresMfa: true, mfaToken };
    }

    // 6. Destroy pre-login session, create fresh one
    await this.sessionStore.destroyIfExists(req.cookies.sessionId);
    const sessionId = await this.createSession(user.id, req);
    res.cookie('sessionId', sessionId, SESSION_COOKIE_OPTIONS);

    // 7. Record login and get previous session info for display
    const lastLogin = await this.recordSuccessfulLogin(user.id, req);

    logger.info('User login successful', { userId: user.id });

    return {
      user:      new UserProfileDto(user),
      lastLogin,
    };
  }
}
```

---

## Gap Detection Table (Additions)

| Gap | What to Look For | Severity |
|---|---|---|
| Auth logic inline in handlers | JWT check or session check inside a service or repository method | 🟠 |
| Auth fails open | `catch (e) { /* ignore */ return user; }` on token verification | 🔴 |
| Admin route less secure than user route | Admin endpoint with no MFA requirement | 🔴 |
| `dto.role` used directly | `userRepo.create({ role: dto.role })` — role from user input | 🔴 |
| Constant-time not used | Timing difference between unknown email and wrong password | 🟠 |
| No email enumeration protection | Register reveals whether email is already taken | 🟠 |
| Email verification not required | Users can access app with unverified email | 🟡 |
| External credentials in source code | DB password, SMTP password in codebase | 🔴 |
| Credentials sent via GET | `?password=` in URL | 🔴 |
| Open redirect after auth | `res.redirect(req.query.next)` without whitelist | 🔴 |
