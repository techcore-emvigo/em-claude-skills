# NodeJS & NestJS — Gap Detection

## NodeJS Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Blocking event loop | `fs.readFileSync`, `execSync`, `JSON.parse` on large payload in request handler | 🟠 |
| New DB connection per request | `new Pool()` or `mongoose.connect()` inside a handler function | 🔴 |
| Unhandled promise rejection | No `process.on("unhandledRejection", ...)` handler | 🟠 |
| No graceful shutdown | No `SIGTERM`/`SIGINT` handler — in-flight requests dropped on deploy | 🟠 |
| `npm install` in CI | Should be `npm ci` — non-deterministic builds | 🟡 |
| Missing `helmet` | No `app.use(helmet())` — missing security headers | 🟠 |
| Missing rate limiter | No `express-rate-limit` or equivalent on auth/public endpoints | 🔴 |

## NestJS Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Logic in controller | Business rules, DB calls, or external API calls inside `@Controller` class | 🟠 |
| No `ValidationPipe` globally | `app.useGlobalPipes(new ValidationPipe(...))` absent in `main.ts` | 🟠 |
| `whitelist: false` on ValidationPipe | Extra fields not stripped — prototype pollution / mass assignment risk | 🔴 |
| `@Injectable()` service doing too much | Service handles multiple unrelated concerns (SRP violation) | 🟠 |
| No `@UseGuards` on protected routes | Missing auth guard on non-public endpoints | 🔴 |
| No exception filter | No `@Catch()` filter — raw errors leak to client | 🟠 |
| Circular module dependency | Module A imports Module B which imports Module A | 🔴 |
| No DTO for request body | Controller accepts `@Body() body: any` without a typed DTO | 🟠 |
| Missing `response_model` equivalent | Returning full entity instead of a DTO — may expose sensitive fields | 🟠 |

## Generation Checklist
- [ ] `ValidationPipe` with `whitelist: true, forbidNonWhitelisted: true, transform: true`
- [ ] `JwtAuthGuard` (or equivalent) applied at router level; `@Public()` for exceptions
- [ ] `@Catch()` exception filter returns consistent error envelope
- [ ] Connections (DB, Redis) initialized at module startup, not per-request
- [ ] `SIGTERM` handler for graceful shutdown
- [ ] Each module owns only its domain — no cross-module repository imports
