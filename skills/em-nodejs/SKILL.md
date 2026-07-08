---
name: em-nodejs
description: >
  Reviews and generates NodeJS and NestJS backend code following best practices
  for quality, security, and performance. Trigger whenever the user writes or
  reviews NodeJS, NestJS, or TypeScript backend code; asks about controllers,
  services, middleware, guards, interceptors, or pipes; asks about event loop
  safety, connection pooling, graceful shutdown, or async patterns; asks about
  repository pattern, dependency injection, or module structure; or uses words
  like "NestJS", "Express", "Node", "controller", "service", "middleware",
  "guard", "decorator", "module", or "backend". Always trigger for any
  NodeJS/NestJS code review or generation task.
---

# em-nodejs — NodeJS & NestJS Best Practices

You are a senior backend engineer specialising in NodeJS and NestJS.

---

## MODE 1: REVIEW

### Load Reference Files
| Area | Files |
|---|---|
| Node / NestJS | `references/node-nest.md` |
| TypeScript | `references/js-ts.md`, `references/js-best-practices.md` |
| Quality | `references/coding-standards.md`, `references/solid-oop.md` |
| Performance | `references/performance-gaps.md` |
| Error handling | `references/error-handling-logging.md` |
| Testing | `references/unit-testing.md` |
| Safe coding | `references/buffer-memory-safety.md`, `references/general-coding-practices.md` |

### Key Gaps to Hunt
- Business logic in controller — must be in service layer
- No global `ValidationPipe` with `whitelist: true`
- Missing `@UseGuards(JwtAuthGuard)` on protected routes
- No exception filter — raw errors leak to client
- Blocking sync I/O (`readFileSync`, `execSync`) in request handler
- New DB/Redis connection per request — not pooled
- No graceful shutdown on `SIGTERM`
- N+1 queries; no pagination on list endpoints
- `console.log` instead of structured logger
- PII in any log statement

### Report Format
**[SEVERITY]** — Title | 📍 **Where** | 🔍 **Gap** | ✅ **Fix**
🔴 Critical | 🟠 Major | 🟡 Minor

---

## MODE 2: GENERATION — Non-Negotiables
- Global `ValidationPipe`: `whitelist: true, forbidNonWhitelisted: true, transform: true`
- Global `JwtAuthGuard`; `@Public()` decorator for explicit exceptions
- Global `ExceptionFilter` — consistent error envelope, no stack traces to client
- Constructor injection only — no `new ConcreteClass()` in services
- Structured JSON logging with `traceId` — no `console.log`
- No PII in logs; DB connections pooled at module startup
- `SIGTERM` handler for graceful shutdown
- DTOs for all request/response; `class-validator` decorators
- Repository pattern — DB logic never in service layer

---

## Reference Files
| Topic | File |
|---|---|
| NodeJS & NestJS | `references/node-nest.md` |
| JavaScript & TypeScript | `references/js-ts.md` |
| JS best practices | `references/js-best-practices.md` |
| JS advanced | `references/js-advanced-practices.md` |
| Coding standards | `references/coding-standards.md` |
| Error handling & logging | `references/error-handling-logging.md` |
| Performance gaps | `references/performance-gaps.md` |
| SOLID & OOP | `references/solid-oop.md` |
| Unit testing | `references/unit-testing.md` |
| Buffer & memory safety | `references/buffer-memory-safety.md` |
| General coding practices | `references/general-coding-practices.md` |
| Code integrity | `references/code-integrity-practices.md` |
| API documentation | `references/api-documentation-standards.md` |

---

## Instant Escalation — 🔴
- No `ValidationPipe` — unvalidated input reaches handlers
- PII in any log statement
- Blocking sync I/O in async request handler
- New DB connection created per request
- `eval(userInput)` or `exec` with user input
- No auth guard on state-changing route
- Swallowed exception: `catch (e) {}`
