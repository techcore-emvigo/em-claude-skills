# .NET (C#) — Gap Detection

## Code Quality Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Missing nullable enable | `<Nullable>` not set to `enable` in `.csproj` | 🟠 |
| `.Result` / `.Wait()` on async | `task.Result`, `task.Wait()` — deadlock risk in async context | 🔴 |
| `catch (Exception e) {}` | Swallowing all exceptions silently | 🟠 |
| `new ConcreteService()` in controller/service | Constructor `new`-ing a dependency instead of injecting | 🟠 |
| Business logic in controller | Calculation, validation rule, or workflow decision in an `[ApiController]` | 🟠 |
| `IConfiguration` injected into business layer | Domain/application service depending on raw config | 🟡 |
| No `CancellationToken` on async method | Public async methods missing CT parameter — can't be cancelled | 🟡 |

## ASP.NET Core Security Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No `[Authorize]` on protected action | Controller action missing auth attribute with no global policy | 🔴 |
| Connection string with password in code | `"Server=...;Password=realpass"` literal in source | 🔴 |
| `AllowAnyOrigin()` with credentials | CORS allowing any origin while using cookie auth | 🔴 |
| Raw SQL string concatenation | `$"SELECT * FROM Users WHERE Id = {id}"` in EF raw query | 🔴 |
| `AllowAnonymous` on sensitive endpoint | Admin action decorated with `[AllowAnonymous]` | 🔴 |
| No `app.UseHttpsRedirection()` | HTTPS redirect middleware missing | 🟠 |
| No `app.UseHsts()` | HSTS header not configured in production | 🟠 |

## EF Core Performance Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Missing `AsNoTracking()` | Read-only query without `AsNoTracking()` — unnecessary change tracking | 🟡 |
| Lazy loading enabled globally | `UseLazyLoadingProxies()` — hides N+1 in production | 🟠 |
| `ToList()` before `Where` | `dbSet.ToList().Where(x => ...)` — loads entire table into memory | 🔴 |
| Missing `.Include()` for nav property | Accessing `order.User` without `.Include(o => o.User)` — N+1 | 🔴 |

## Generation Checklist
- [ ] `<Nullable>enable</Nullable>` in every project file
- [ ] Constructor injection only; no `new` inside services
- [ ] `CancellationToken` on all public async methods
- [ ] `AsNoTracking()` on all read-only EF queries
- [ ] `IOptions<T>` for typed config; no raw `IConfiguration` in business layer
- [ ] `[Authorize]` at controller level with `[AllowAnonymous]` on explicit exceptions
