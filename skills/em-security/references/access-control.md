# Access Control — Secure Coding Reference

## Core Principle
Authorization decisions must always be made server-side, through a single centralised
component, on every request — without exception. Access control must fail securely:
when in doubt, deny.

---

## 1. Centralised Authorization Component

Never scatter auth checks across controllers and services. One guard, one policy engine.

```typescript
// ❌ Bad — auth logic duplicated and inconsistent across handlers
@Get('orders/:id')
async getOrder(@Param('id') id: string, @Req() req: Request) {
  if (req.headers.authorization) { // inconsistent, error-prone
    return this.orderRepo.findById(id);
  }
}

@Get('invoices/:id')
async getInvoice(@Param('id') id: string, @Req() req: Request) {
  const token = req.cookies.session; // different check — inconsistent!
  if (token) {
    return this.invoiceRepo.findById(id);
  }
}

// ✅ Good — single JwtAuthGuard applied globally; @Public() for exceptions
// main.ts — apply globally so nothing is accidentally unprotected
app.useGlobalGuards(new JwtAuthGuard());

// auth/jwt-auth.guard.ts — single site-wide component
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.get<boolean>('isPublic', context.getHandler());
    if (isPublic) return true; // explicitly marked public — allowed
    return super.canActivate(context); // everything else requires auth
  }

  // ✅ Fail securely — deny access on any auth error
  handleRequest(err: Error, user: AuthUser) {
    if (err || !user) throw new UnauthorizedException('Authentication required');
    return user;
  }
}

// ✅ Explicit opt-out for truly public endpoints
@Get('products')
@Public()  // intentional — documented decision to allow unauthenticated access
async listProducts() { return this.productService.list(); }
```

---

## 2. Role-Based Access Control (RBAC)

```typescript
// ✅ Good — RBAC with roles from server-side JWT only
// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<UserRole[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles?.length) return true;

    const { user } = context.switchToHttp().getRequest<Request>();

    // ✅ Role comes from verified JWT payload — never from request body
    const hasRole = requiredRoles.some(role => user.roles?.includes(role));

    if (!hasRole) {
      logger.warn('Access denied — insufficient role', {
        userId:        user.id,
        requiredRoles,
        userRoles:     user.roles,
        path:          context.switchToHttp().getRequest().path,
      });
    }
    return hasRole;
  }
}

// ✅ Usage — role required at route level
@Delete('admin/users/:id')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.ADMIN)
async deleteUser(@Param('id') id: string, @CurrentUser() actor: AuthUser) {
  logger.info('Admin deleted user', { actorId: actor.id, targetUserId: id });
  return this.userService.delete(id);
}
```

---

## 3. Direct Object Reference (IDOR) Protection

Every resource access must verify the requesting user owns or has permission for that resource.

```typescript
// ❌ Critical — IDOR: any authenticated user can read any order by guessing the ID
@Get('orders/:id')
@UseGuards(JwtAuthGuard)
async getOrder(@Param('id') id: string) {
  return this.orderRepo.findById(id); // no ownership check!
}

// ✅ Good — ownership verified before returning resource
@Get('orders/:id')
@UseGuards(JwtAuthGuard)
async getOrder(
  @Param('id', ParseUUIDPipe) id: string,
  @CurrentUser() user: AuthUser,
): Promise<OrderResponse> {
  const order = await this.orderRepo.findById(id);

  if (!order) throw new NotFoundException('Order not found');

  // Admin can see all; others only see their own
  if (!user.roles.includes(UserRole.ADMIN) && order.userId !== user.id) {
    // ✅ Log access control failure
    logger.warn('IDOR attempt blocked', {
      userId:       user.id,
      resourceType: 'order',
      resourceId:   id,
      resourceOwner: order.userId,
    });
    // ✅ Return 404 not 403 — don't confirm the resource exists to unauthorised user
    throw new NotFoundException('Order not found');
  }

  return new OrderResponse(order);
}

// ✅ Good — repository method scoped to userId
async getMyOrders(userId: string): Promise<Order[]> {
  // DB query always includes userId — not just in application layer
  return this.prisma.order.findMany({
    where: { userId, isDeleted: false },
  });
}
```

---

## 4. Privileged Logic Segregation

```typescript
// ❌ Bad — admin logic mixed with user logic
class UserService {
  async updateUser(id: string, dto: UpdateUserDto, requesterId: string) {
    if (dto.role === 'admin') {          // admin check buried in regular service
      await this.promoteToAdmin(id);
    }
    return this.userRepo.update(id, dto); // dto.role could bypass check!
  }
}

// ✅ Good — separate admin service; regular service cannot touch privileged fields
// services/user.service.ts — for regular users
class UserService {
  async updateProfile(userId: string, dto: UpdateProfileDto): Promise<User> {
    // UpdateProfileDto only has: name, phone, avatar — NO role, NO isActive
    return this.userRepo.update(userId, {
      name:   dto.name,
      phone:  dto.phone,
      avatar: dto.avatar,
    });
  }
}

// services/admin-user.service.ts — for admin operations only
class AdminUserService {
  async updateUserRole(targetId: string, role: UserRole, actorId: string): Promise<void> {
    logger.info('Role change', { actorId, targetId, newRole: role });
    await this.auditService.log({ actorId, action: 'user.role_changed', resourceId: targetId });
    await this.userRepo.updateRole(targetId, role);
    await this.sessionStore.revokeAll(targetId); // invalidate sessions after role change
  }
}
```

---

## 5. Fail Securely — Deny on Error

```typescript
// ❌ Bad — auth check fails open (grants access on error)
async canAccess(userId: string, resourceId: string): Promise<boolean> {
  try {
    const policy = await this.policyService.get(userId, resourceId);
    return policy.allowed;
  } catch (e) {
    return true; // ❌ ERROR: grants access if policy service is down!
  }
}

// ✅ Good — fail closed: deny access if security configuration unavailable
async canAccess(userId: string, resourceId: string): Promise<boolean> {
  try {
    const policy = await this.policyService.get(userId, resourceId);
    return policy?.allowed === true; // explicitly true — undefined = deny
  } catch (error) {
    // If we can't check the policy, DENY and log — never grant on uncertainty
    logger.error('Access control check failed — denying access', {
      userId, resourceId, error: error.message,
    });
    return false; // ✅ Fail securely
  }
}
```

---

## 6. Transaction Rate Limiting Per User

```typescript
// ✅ Good — per-user transaction limit to deter automated attacks
// rate-limiter.service.ts
@Injectable()
export class UserRateLimiter {

  async checkLimit(
    userId:     string,
    action:     string,
    maxPerHour: number,
  ): Promise<void> {
    const key   = `ratelimit:${action}:${userId}:${getHourBucket()}`;
    const count = await redis.incr(key);
    await redis.expire(key, 3600);

    if (count > maxPerHour) {
      logger.warn('User rate limit exceeded', { userId, action, count, maxPerHour });
      throw new TooManyRequestsException(
        `Too many ${action} attempts. Try again later.`
      );
    }
  }
}

// ✅ Usage — apply per-user limits on sensitive operations
@Post('payments/transfer')
@UseGuards(JwtAuthGuard)
async transfer(@Body() dto: TransferDto, @CurrentUser() user: AuthUser) {
  // Business allows 50 transfers/hour; limit to 100 to deter automation
  await this.rateLimiter.checkLimit(user.id, 'payment_transfer', 100);
  return this.paymentService.transfer(dto, user.id);
}
```

---

## 7. Periodic Re-validation of Long Sessions

```typescript
// ✅ Good — re-validate user privileges on long-lived sessions
// Catches role changes, account disabling, employment termination mid-session
@Injectable()
export class SessionRevalidationMiddleware implements NestMiddleware {
  private REVALIDATION_INTERVAL_MS = 5 * 60 * 1000; // every 5 minutes

  async use(req: Request, res: Response, next: NextFunction) {
    const session = req.userSession;
    if (!session) return next();

    const lastValidated = session.lastValidatedAt?.getTime() ?? 0;
    const needsRevalidation = Date.now() - lastValidated > this.REVALIDATION_INTERVAL_MS;

    if (needsRevalidation) {
      const user = await this.userRepo.findById(session.userId);

      // Account deleted, disabled, or role changed — force logout
      if (!user || !user.isActive) {
        await this.sessionStore.destroy(session.id);
        res.clearCookie('sessionId');
        return res.status(401).json({ error: { code: 'SESSION_INVALIDATED' } });
      }

      // Update session with latest role/permission snapshot
      await this.sessionStore.update(session.id, {
        roles:           user.roles,
        lastValidatedAt: new Date(),
      });
    }
    next();
  }
}
```

---

## 8. Account Auditing — Disable Unused Accounts

```typescript
// ✅ Good — scheduled job to disable accounts unused for 30 days
// jobs/account-audit.job.ts
@Cron('0 2 * * *') // daily at 2 AM
async disableInactiveAccounts(): Promise<void> {
  const INACTIVE_THRESHOLD_DAYS = 30;
  const cutoff = new Date(Date.now() - INACTIVE_THRESHOLD_DAYS * 24 * 60 * 60 * 1000);

  const inactiveUsers = await this.userRepo.findInactiveSince(cutoff);

  for (const user of inactiveUsers) {
    await this.userRepo.disable(user.id);
    await this.sessionStore.revokeAll(user.id);
    await this.auditService.log({
      action:     'account.auto_disabled',
      resourceId: user.id,
      metadata:   { reason: 'inactivity', lastActiveAt: user.lastActiveAt },
    });
    logger.info('Account auto-disabled due to inactivity', { userId: user.id });
  }
}

// ✅ Good — immediately revoke all sessions when account disabled
async disableAccount(userId: string, reason: string, actorId: string): Promise<void> {
  await this.userRepo.update(userId, { isActive: false, disabledAt: new Date() });
  await this.sessionStore.revokeAll(userId);  // kill all active sessions immediately
  await this.auditService.log({
    actorId, action: 'account.disabled', resourceId: userId,
    metadata: { reason },
  });
  logger.info('Account disabled', { actorId, userId, reason });
}
```

---

## 9. Service Account Least Privilege

```typescript
// ✅ Good — separate DB credentials per role/trust level
// config/database.config.ts
export const DB_CONNECTIONS = {
  // Application user: DML only — SELECT, INSERT, UPDATE, DELETE
  app: {
    url:      process.env.DB_APP_URL,
    // GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
  },
  // Read-only user: for reporting and analytics queries
  readonly: {
    url:      process.env.DB_READONLY_URL,
    // GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
  },
  // Migration user: DDL only, used by deploy pipeline only
  migration: {
    url:      process.env.DB_MIGRATION_URL,
    // GRANT ALL ON ALL TABLES IN SCHEMA public TO migration_user;
    // — never used at runtime
  },
};

// ✅ Good — service account for external integrations has minimum permissions
// In AWS IAM policy for the orders-service Lambda:
const ordersServicePolicy = {
  Version: '2012-10-17',
  Statement: [
    {
      Effect:   'Allow',
      Action:   ['sqs:ReceiveMessage', 'sqs:DeleteMessage', 'sqs:GetQueueAttributes'],
      Resource: `arn:aws:sqs:${region}:${account}:orders-queue`, // specific queue only
    },
    {
      Effect:   'Allow',
      Action:   ['secretsmanager:GetSecretValue'],
      Resource: `arn:aws:secretsmanager:${region}:${account}:secret:orders-service/*`,
    },
    // NOT: Action: '*' Resource: '*' — over-permissioned
  ],
};
```

---

## Gap Detection Table

| Gap | What to Look For | Severity |
|---|---|---|
| Auth check not applied globally | `@UseGuards(JwtAuthGuard)` missing on routes; no global guard | 🔴 |
| Role from request body | `if (req.body.role === 'admin')` instead of JWT claim | 🔴 |
| No ownership check on resource | `repo.findById(id)` with no `userId` comparison | 🔴 |
| 403 instead of 404 on IDOR | Returns Forbidden (confirms resource exists) — should be 404 | 🟠 |
| Auth fails open on error | `catch (e) { return true; }` in authorization check | 🔴 |
| No per-user rate limiting | Operations with no user-scoped transaction counter | 🟠 |
| Admin logic in regular service | `if (dto.isAdmin)` in a non-admin service method | 🟠 |
| No session re-validation | Long sessions with no periodic privilege re-check | 🟠 |
| Account disable doesn't kill sessions | `user.isActive = false` without `sessionStore.revokeAll()` | 🔴 |
| Service account over-privileged | `Action: '*'` or `Resource: '*'` in IAM policy | 🔴 |
| Single DB user for all operations | One DB connection string for migrations, app, and reporting | 🟠 |
| Unused accounts not disabled | No automated job to disable accounts inactive > 30 days | 🟠 |
| `Referer` header used for auth | `if (req.headers.referer === 'trusted.com')` as access check | 🔴 |
| No access control audit log | Role changes, account disabling with no audit entry | 🟠 |
| State stored on client unencrypted | User role/permissions in JWT without signature verification | 🔴 |
