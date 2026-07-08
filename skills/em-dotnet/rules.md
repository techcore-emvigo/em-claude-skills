# em-dotnet — Rules Reference

## 1. Code Style & Structure
| Rule | Description | Severity |
|---|---|---|
| DN-01 | `<Nullable>enable</Nullable>` in every `.csproj` | 🟠 |
| DN-02 | `PascalCase` methods/properties; `camelCase` locals/params; `_camelCase` private fields | 🟡 |
| DN-03 | Constructor injection only — no `new ConcreteService()` inside services or controllers | 🟠 |
| DN-04 | Business logic in application/service layer — never in controllers | 🟠 |
| DN-05 | `IOptions<T>` for typed config — no raw `IConfiguration` in business layer | 🟡 |
| DN-06 | `CancellationToken` parameter on all public async methods | 🟡 |
| DN-07 | Methods ≤ 30 lines; classes ≤ 200 lines | 🟠 |

## 2. Async & Performance
| Rule | Description | Severity |
|---|---|---|
| DN-08 | No `.Result` or `.Wait()` on async tasks — deadlock risk | 🔴 |
| DN-09 | `async/await` all the way — never mix sync and async | 🟠 |
| DN-10 | `HttpClientFactory` always — never `new HttpClient()` | 🔴 |
| DN-11 | All list endpoints paginated — cursor or page-based | 🔴 |
| DN-12 | All external calls have timeout and `CancellationToken` | 🟠 |

## 3. Entity Framework Core
| Rule | Description | Severity |
|---|---|---|
| DN-13 | `AsNoTracking()` on all read-only queries | 🟡 |
| DN-14 | No lazy loading in production — explicit `Include()`/`ThenInclude()` | 🟠 |
| DN-15 | No `ToList()` before `Where()` — filter at DB level | 🔴 |
| DN-16 | Migrations reviewed before applying — `EXPLAIN` plan checked | 🟠 |
| DN-17 | No raw SQL string concatenation — `FromSqlRaw` with parameters or LINQ | 🔴 |

## 4. Security
| Rule | Description | Severity |
|---|---|---|
| DN-18 | `[Authorize]` at controller level; `[AllowAnonymous]` explicit on public actions | 🔴 |
| DN-19 | Role from JWT claims only — never from request body | 🔴 |
| DN-20 | No connection string in source — from secrets manager or environment | 🔴 |
| DN-21 | CORS: explicit `WithOrigins(...)` — never `AllowAnyOrigin()` with credentials | 🔴 |
| DN-22 | Input validated with `[ApiController]` + FluentValidation or DataAnnotations | 🟠 |

## 5. Error Handling & Logging
| Rule | Description | Severity |
|---|---|---|
| DN-23 | No `catch (Exception e) {}` — swallowing exceptions | 🔴 |
| DN-24 | `ProblemDetails` (RFC 7807) for all error responses | 🟠 |
| DN-25 | No stack traces or DB error details in API responses | 🟠 |
| DN-26 | No PII in any log statement | 🔴 |
| DN-27 | Structured logging with `traceId` — no `Console.Write` | 🟠 |

## 6. Testing
| Rule | Description | Severity |
|---|---|---|
| DN-28 | xUnit + FluentAssertions + Moq/NSubstitute for unit tests | 🟠 |
| DN-29 | `WebApplicationFactory<Program>` for integration tests | 🟠 |
| DN-30 | `Testcontainers` for real DB tests — not H2 / in-memory fakes | 🟠 |
| DN-31 | Coverage ≥ 80% on service layer; ≥ 90% on domain logic | 🟠 |

## 7. Instant Escalation — 🔴
| # | Violation |
|---|---|
| ESC-01 | `.Result` or `.Wait()` on any async task |
| ESC-02 | `new HttpClient()` inside a handler or service |
| ESC-03 | EF Core `ToList()` before `Where()` — full table loaded |
| ESC-04 | Raw SQL with string concatenation |
| ESC-05 | Connection string hardcoded in source |
| ESC-06 | PII in any log statement |
| ESC-07 | No `[Authorize]` on state-changing action |
| ESC-08 | `catch (Exception e) {}` swallowing all exceptions |
