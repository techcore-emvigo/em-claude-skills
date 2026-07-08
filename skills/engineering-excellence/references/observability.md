# Logging, Observability & Audit — Gap Detection

## Logging Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| PII in logs | `email`, `phone`, `ssn`, `password`, `token` in log statements | 🔴 |
| `console.log` / `print` in production | Debug statements as the primary logging mechanism | 🟡 |
| No structured (JSON) logging | Log messages as plain strings — not machine-parseable | 🟠 |
| Missing `trace_id` / `correlation_id` | Log lines with no request correlation — can't trace a request | 🟠 |
| Missing `service` / `environment` fields | Logs from different services indistinguishable in aggregator | 🟠 |
| Error logged without stack trace | `logger.error(err.message)` — stack lost, hard to debug | 🟠 |
| Wrong log level | Using `ERROR` for expected user errors (validation); using `INFO` for everything | 🟡 |
| No log on important operations | Payment processed, user registered, order shipped — no log entry | 🟠 |

## Metrics & Alerting Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No RED metrics | No request rate, error rate, or duration tracking per service | 🟠 |
| No DB query latency tracking | Slow queries invisible until users complain | 🟠 |
| No external call latency tracking | Third-party API latency/failures not instrumented | 🟠 |
| No queue consumer lag alerting | Kafka/SQS consumer falling behind with no alert | 🟠 |
| Alerting on raw count not rate | Alert fires on "100 errors" not "error rate > 1%" — too noisy/too quiet | 🟡 |
| No dead man's switch | Scheduled job has no alert if it fails to run | 🟠 |

## Distributed Tracing Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No trace ID propagation | Service creates new `trace_id` instead of reading from incoming request | 🟠 |
| Trace not propagated to queue | Message published without `traceparent` / `correlationId` header | 🟠 |
| No spans on DB calls | DB queries not instrumented — latency invisible in traces | 🟡 |
| No spans on external HTTP calls | Outbound API calls not wrapped in spans | 🟡 |

## Audit Trail Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No audit log on sensitive mutations | User role change, payment refund, account deletion with no audit entry | 🔴 |
| Audit log in same DB as app data | Audit log can be modified by same DB user as application | 🟠 |
| No `actor_id` in audit entry | Audit log records what changed but not who changed it | 🔴 |
| No retention policy | Audit logs deleted or overwritten — compliance violation | 🟠 |
| Auth events not logged | Login, logout, failed login, MFA challenge not recorded | 🟠 |

## Health Check Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No `/health` endpoint | Service with no liveness check | 🟠 |
| `/health` checks DB/dependencies | Liveness endpoint fails when DB is down — restarts healthy pod | 🟠 |
| No `/ready` endpoint | No readiness check — traffic sent to pod before it's ready | 🟠 |
| Health endpoint exposed publicly | `/health` reveals internal dependency status to internet | 🟡 |

## Generation Checklist
- [ ] Structured JSON logs with `timestamp`, `level`, `service`, `trace_id`, `message`
- [ ] No PII in any log statement — log user ID, not email/name
- [ ] `trace_id` read from incoming request headers and included on all log lines
- [ ] RED metrics instrumented at middleware level
- [ ] Audit log for all sensitive mutations: `actor_id`, `action`, `resource_id`, `timestamp`
- [ ] `/health` (liveness — process only) and `/ready` (readiness — with deps) endpoints
- [ ] DLQ depth and consumer lag alerts configured

---

## 5. Pre-Release Observability Checklist

| Gap | What to Look For | Severity |
|---|---|---|
| Sentry (or APM) not integrated | Production errors go undetected | 🔴 |
| New errors in this release not reviewed in Sentry | Known errors shipped intentionally | 🔴 |
| Console errors/warnings not cleared before release | Silent JavaScript failures in production | 🟠 |
| Third-party API request/response not logged | Integration issues invisible in production | 🟠 |
| Logs not centralised (ELK, Datadog, CloudWatch, Loki) | Distributed logs, no unified view | 🟠 |
| Logs not encrypted at rest | Sensitive data exposed in plaintext log files | 🟠 |
| Reports (quality, performance) not integrated into CI/CD | Quality gates bypassable | 🟠 |
| Audit trails not maintained for config changes and deployments | Can't trace what changed and when | 🟠 |
| Error trend reports not generated for critical errors over time | Recurring issues not surfaced | 🟡 |
| Log review process not in place for suspicious activity | Attacks undetected | 🟠 |
| Key findings from reports not shared with development team | Issues siloed, team doesn't learn | 🟡 |

---

## 6. Monitoring & Logging Tools — Standards

Every project and its components must have monitoring and logging tools integrated.
Data must flow into these tools properly — not just installed but actively populated.

### Mandatory Tool Stack

| Tool | Purpose | Scope |
|---|---|---|
| **LogDNA / Mezmo** | Centralised log aggregation and search | All services, backend and frontend |
| **Sentry** | Error tracking, performance, session replay | All frontend (web + mobile) + backend |
| **New Relic / Datadog** | APM, infrastructure monitoring, alerting | All production services |
| **SonarQube** | Static code analysis, security hotspots | All repositories |

```typescript
// ✅ Good — LogDNA integration (Node.js backend)
// Install: npm install @logdna/logger
import { createLogger } from '@logdna/logger';

const ldLogger = createLogger(process.env.LOGDNA_INGESTION_KEY, {
  app:      process.env.SERVICE_NAME,
  env:      process.env.NODE_ENV,
  tags:     ['backend', process.env.SERVICE_NAME],
  hostname: process.env.HOSTNAME,
});

// ✅ Wrap as your standard logger — same structured interface
export const logger = {
  info:  (msg: string, meta?: object) => ldLogger.info(msg,  { meta }),
  warn:  (msg: string, meta?: object) => ldLogger.warn(msg,  { meta }),
  error: (msg: string, meta?: object) => ldLogger.error(msg, { meta }),
  debug: (msg: string, meta?: object) => ldLogger.debug(msg, { meta }),
};
```

### Log Level Management Per Environment

Different environments require different verbosity. Configure at startup, never hardcode.

```typescript
// config/logger.config.ts
export const LOG_LEVEL_BY_ENV: Record<string, string> = {
  production:  'warn',    // warn + error only — info is too verbose for prod volume
  staging:     'info',    // full info + warn + error for debugging
  development: 'debug',   // everything — verbose for local development
  test:        'silent',  // no logs during automated tests
};

export const LOG_LEVEL = LOG_LEVEL_BY_ENV[process.env.NODE_ENV ?? 'development'];

// ✅ Usage — in every service, one consistent logger instance
import pino from 'pino';
import { LOG_LEVEL } from '../config/logger.config';

export const logger = pino({
  level: LOG_LEVEL,
  base:  { service: process.env.SERVICE_NAME, env: process.env.NODE_ENV },
});
```

### Log Level Decision Guide

```typescript
// ❌ Bad — everything logged at ERROR level
logger.error('User fetched');             // not an error!
logger.error('Validation failed');        // expected user error — use warn

// ❌ Bad — errors logged at INFO level
logger.info('Payment charge failed');     // this needs alerting — use error!

// ✅ Good — correct level per scenario
logger.debug({ query, params }, 'SQL executed.');                    // dev detail only
logger.info ({ userId, action: 'login' }, 'User logged in.');       // lifecycle event
logger.warn ({ attempt, maxAttempts }, 'Retry attempt.');           // degraded, recoverable
logger.error({ err, orderId }, 'Payment processing failed.');       // unexpected failure
```

### New Relic / Datadog APM Integration

```typescript
// ✅ Good — New Relic auto-instrumentation (add at application entry point)
// Placed at the VERY TOP of main.ts / index.ts — before any other imports
require('newrelic'); // or: import 'newrelic'

// Custom attributes for better filtering in New Relic dashboards
const newrelic = require('newrelic');

app.use((req, res, next) => {
  newrelic.addCustomAttributes({
    userId:    req.user?.id,
    requestId: req.headers['x-request-id'],
    service:   process.env.SERVICE_NAME,
  });
  next();
});
```

```typescript
// ✅ Good — Google PageSpeed Insights threshold enforcement
// In CI/CD — fail deployment if PageSpeed score drops below 80
// Use lighthouse-ci:
// .lighthouserc.json
{
  "ci": {
    "assert": {
      "assertions": {
        "categories:performance":   ["error", { "minScore": 0.80 }],
        "categories:accessibility": ["error", { "minScore": 0.90 }],
        "categories:best-practices":["warn",  { "minScore": 0.85 }],
        "categories:seo":           ["warn",  { "minScore": 0.80 }]
      }
    }
  }
}
```

---

## Gap Detection Table (Additions)

| Gap | What to Look For | Severity |
|---|---|---|
| LogDNA / log aggregator not integrated | Logs only in local container stdout — no centralised search | 🟠 |
| Sentry not in a frontend project | No `@sentry/react`, `@sentry/nextjs`, `@sentry/react-native` | 🔴 |
| New Relic / Datadog APM not configured | No APM in production services — latency invisible | 🟠 |
| SonarQube not integrated in repo | No `sonar-project.properties` or pipeline scan step | 🟠 |
| Log level same in all environments | `level: 'debug'` in production — log volume/cost problem | 🟠 |
| No log level configuration by environment | Hardcoded `console.log` statements, no level management | 🟠 |
| Expected errors logged at ERROR level | `logger.error('Validation failed')` — use `warn` for user errors | 🟡 |
| Unexpected failures logged at INFO level | `logger.info('DB connection failed')` — use `error` | 🟠 |
| Google PageSpeed not checked | No Lighthouse CI in pipeline; score not verified before release | 🟠 |
