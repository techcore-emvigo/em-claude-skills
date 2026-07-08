# Security Event Logging & Secure Error Responses

Covers: mandatory security event logging (auth attempts, access failures, admin
actions, TLS failures), safe error responses, log injection prevention.

---

## 5. Security Event Logging — What MUST Be Logged

Every security-relevant event must be logged with enough context to support incident response.
Use the centralised logger — never scattered `console.log` calls.

```typescript
// ✅ Good — centralised security event logger
// logging/security-logger.ts
export class SecurityLogger {

  // 1. Log ALL authentication attempts (success and failure)
  static authAttempt(email: string, success: boolean, context: SecurityContext): void {
    const level = success ? 'info' : 'warn';
    logger[level]('Authentication attempt', {
      event:        success ? 'auth.success' : 'auth.failure',
      maskedEmail:  mask.email(email),
      ipAddress:    context.ip,
      userAgent:    context.userAgent,
      traceId:      context.traceId,
    });
  }

  // 2. Log ALL input validation failures
  static validationFailure(fields: Record<string, string>, context: SecurityContext): void {
    logger.warn('Input validation failed', {
      event:   'validation.failure',
      fields:  Object.keys(fields),  // field names only — not values (may contain PII)
      path:    context.path,
      method:  context.method,
      traceId: context.traceId,
    });
  }

  // 3. Log ALL access control failures
  static accessDenied(
    userId: string,
    resource: string,
    reason: string,
    context: SecurityContext
  ): void {
    logger.warn('Access control failure', {
      event:    'access.denied',
      userId,
      resource,
      reason,
      path:     context.path,
      traceId:  context.traceId,
    });
  }

  // 4. Log attempts with invalid/expired session tokens
  static invalidSession(token: string, reason: string, context: SecurityContext): void {
    logger.warn('Invalid session token attempt', {
      event:       'session.invalid',
      tokenPrefix: token.substring(0, 8) + '...', // partial only — never full token
      reason,
      ipAddress:   context.ip,
      traceId:     context.traceId,
    });
  }

  // 5. Log ALL admin actions / security config changes
  static adminAction(
    actorId:  string,
    action:   string,
    resource: string,
    changes:  Record<string, unknown>
  ): void {
    logger.info('Admin action performed', {
      event:    'admin.action',
      actorId,
      action,
      resource,
      changes,  // what changed — no sensitive values
    });
  }

  // 6. Log state tampering events
  static tamperingDetected(description: string, context: SecurityContext): void {
    logger.error('Potential tampering detected', {
      event:     'security.tampering',
      description,
      ipAddress: context.ip,
      userId:    context.userId,
      traceId:   context.traceId,
    });
  }

  // 7. Log TLS / backend connection failures
  static tlsFailure(endpoint: string, error: Error, context: SecurityContext): void {
    logger.error('TLS connection failure', {
      event:    'tls.failure',
      endpoint,
      error:    error.message,
      traceId:  context.traceId,
    });
  }
}
```

---

## 6. Secure Error Responses — No Information Disclosure

```typescript
// ❌ Bad — leaks system details in error response
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  res.status(500).json({
    error:   err.message,        // may contain DB table names, file paths
    stack:   err.stack,          // reveals code structure
    query:   err['query'],       // SQL query exposed!
    detail:  err['detail'],      // PostgreSQL error detail — schema info
  });
});

// ✅ Good — generic messages in production; full detail only in logs
// filters/global-exception.filter.ts
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx      = host.switchToHttp();
    const res      = ctx.getResponse<Response>();
    const req      = ctx.getRequest<Request>();
    const traceId  = req.traceId ?? generateTraceId();
    const isHttpEx = exception instanceof HttpException;

    const status  = isHttpEx ? exception.getStatus() : 500;
    const isServer = status >= 500;

    // Full detail goes to logs only — never to the client
    if (isServer) {
      logger.error('Unhandled exception', {
        traceId,
        path:    req.path,
        method:  req.method,
        error:   exception instanceof Error ? exception.message : String(exception),
        stack:   exception instanceof Error ? exception.stack : undefined,
      });
    }

    // Client receives only: status code, error code, trace ID
    // NEVER: stack trace, DB errors, file paths, internal IDs
    res.status(status).json({
      error: {
        code:     isHttpEx ? (exception.getResponse() as any)?.error ?? 'HTTP_ERROR'
                           : 'INTERNAL_ERROR',
        message:  isServer
                    ? 'An unexpected error occurred. Please try again.'  // generic for 5xx
                    : (isHttpEx ? exception.message : 'Request failed'), // specific for 4xx
        traceId,  // include trace ID so user can report and you can look up the full log
      },
    });
  }
}
```

---

## 7. Log Injection Prevention

```typescript
// ❌ Bad — user input written directly to logs — attacker can inject fake log entries
logger.info(`User search query: ${req.query.search}`);
// If search = "normal
INFO: Admin logged in as root" — fake log entry injected!

// ✅ Good — structured logging prevents injection (JSON doesn't execute log syntax)
logger.info('User search', {
  query:   req.query.search,   // value is a JSON string field — not part of log format
  userId:  req.user.id,
  traceId: req.traceId,
});
// In JSON: {"level":"info","message":"User search","query":"normal\nINFO: Admin..."}
// The newline is escaped inside the JSON value — log viewers show it correctly

// ✅ Good — strip newlines when writing to legacy line-based log systems
function sanitizeForLog(value: string): string {
  return value
    .replace(/[
]/g, ' ')    // replace newlines with space
    .replace(/	/g,     ' ')    // replace tabs
    .substring(0, 500);         // truncate very long values
}
```

---

## Gap Detection Table (Security-Specific Additions)

| Gap | What to Look For | Severity |
|---|---|---|
| Stack trace in response | `err.stack`, `error.toString()` in API response body | 🔴 |
| DB error detail in response | `err.detail`, `err.query`, PostgreSQL/MySQL error in response | 🔴 |
| Auth failure not logged | Login endpoint with no `logger.warn` on failure | 🟠 |
| Validation failure not logged | Schema validation error with no security log entry | 🟠 |
| Access denied not logged | 403/404 with no log entry | 🟠 |
| Admin action not logged | Role change, config change, account disable with no audit log | 🔴 |
| TLS failure swallowed silently | `catch (e) {}` on outbound HTTPS call — no log | 🟠 |
| Log injection risk | User input string-concatenated into log message (not structured) | 🟠 |
| Session ID in log | `logger.info({ sessionId })` or `console.log(token)` | 🔴 |
| No trace ID in error response | `500` error with no `traceId` — user cannot report the issue | 🟡 |
