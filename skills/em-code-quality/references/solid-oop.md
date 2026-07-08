# SOLID Principles & OOP — Gap Detection

Use this file to identify design quality gaps. Every item below is a concrete smell to look for.

---

## SOLID Violation Detector

### S — Single Responsibility (SRP)
**The smell:** A class/function has more than one reason to change.

| Gap Pattern | How to Spot It | Fix |
|---|---|---|
| God class | Class > 300 lines, has 5+ injected deps, has mixed concerns | Split into focused classes |
| Fat controller | Controller contains business logic, SQL, or external API calls | Extract to service layer |
| Fat service | Service handles multiple unrelated operations (auth AND email AND billing) | Split by domain concern |
| Multi-responsibility function | Function name uses "and": `validateAndSave`, `fetchAndFormat` | Split into two functions |
| Mixed abstraction levels | Function mixes high-level orchestration with low-level implementation details | Extract helper functions |

### O — Open/Closed (OCP)
**The smell:** Adding new behaviour requires modifying existing, tested code.

| Gap Pattern | How to Spot It | Fix |
|---|---|---|
| Growing if/else or switch on type | `if type === 'email'... else if type === 'sms'...` with 3+ branches that grow | Strategy pattern / polymorphism |
| Type check instead of polymorphism | `if (animal instanceof Dog)` to decide behaviour | Override method in subclass / interface |
| Feature flags hardcoded | `if (featureVersion === 2)` scattered through business logic | Inject strategy / use feature abstraction |

### L — Liskov Substitution (LSP)
**The smell:** A subclass breaks the contract of its parent.

| Gap Pattern | How to Spot It | Fix |
|---|---|---|
| `throw NotImplementedException` in override | Subclass overrides method just to throw | Use separate interface instead of inheritance |
| Override that weakens behaviour | Subclass override does less than parent promises | Redesign hierarchy or use composition |
| Subclass that tightens preconditions | Subclass requires more than parent (extra validation, fewer accepted values) | Extract common interface |

### I — Interface Segregation (ISP)
**The smell:** Clients implement methods they don't use.

| Gap Pattern | How to Spot It | Fix |
|---|---|---|
| Fat interface | Interface with 8+ methods used by clients that only need 2-3 | Split into smaller focused interfaces |
| Empty method implementations | `public void SomeMethod() {}` — required by interface but irrelevant | Split interface |
| Forced dependency on unused methods | Class imports a dependency purely for one method on a large interface | Extract minimal interface |

### D — Dependency Inversion (DIP)
**The smell:** High-level code depends on low-level concrete implementations.

| Gap Pattern | How to Spot It | Fix |
|---|---|---|
| `new ConcreteClass()` inside a service | `this.repo = new PostgresUserRepository()` in a service constructor | Inject interface via constructor |
| Domain imports infrastructure | Domain entity imports `@Entity()`, `@Column()` ORM decorators directly | Separate persistence model from domain model |
| Service imports framework type | Business logic imports `Request`, `Response`, `HttpException` from Express/NestJS | Use DTOs; framework types stay in controllers |
| Hardwired external client | `this.stripe = new Stripe(key)` inside a service | Inject `IPaymentGateway` interface |

---

## OOP Design Smells — Quick Reference

| Smell | How to Spot | Severity | Fix |
|---|---|---|---|
| **Anemic Domain Model** | Entities are just data bags (getters/setters only); all logic is in services | 🟠 | Move logic into the entity where it belongs |
| **Primitive Obsession** | Using `string` for email/status, `number` for money, `string[]` for tags | 🟡 | Create Value Objects (`Email`, `Money`, `Tag`) |
| **Feature Envy** | Method uses another class's data more than its own | 🟡 | Move method to the class it envies |
| **Long Parameter List** | Function with 5+ parameters | 🟡 | Introduce a Parameter Object / DTO |
| **Shotgun Surgery** | One business change requires edits in 5+ files | 🟠 | Consolidate related logic; apply SRP |
| **Refused Bequest** | Subclass ignores or overrides most of the parent's methods | 🟠 | Replace inheritance with composition |
| **Speculative Generality** | Abstract base classes, generics, or hooks with only one implementation | 🟡 | Delete until a second use case exists |
| **Inheritance for Code Reuse** | Extends a class just to borrow one utility method | 🟠 | Extract shared logic to a utility / inject as dependency |
| **Mutable Shared State** | Global variable or singleton with mutable state accessed across threads/requests | 🔴 | Make stateless or scope state to request context |

---

## Architectural Layer Violations

These are quality gaps that span architecture and code design:

| Gap | What to Look For | Severity |
|---|---|---|
| Business logic in controller | Validation rules, calculations, or workflow decisions inside a route handler | 🟠 |
| SQL in service layer | Raw `db.query()` calls inside a service — should be in repository | 🟠 |
| HTTP concerns in domain | Domain entity or service imports `Request`, throws `HttpException`, uses status codes | 🟠 |
| Circular layer dependency | `domain/` imports from `infrastructure/` or `api/` | 🔴 |
| Cross-service DB access | Service A's code queries Service B's database tables directly | 🔴 |
| Controller calls repository directly | Controller bypasses service layer, calls repository | 🟠 |
| Domain event published in repository | Repository publishes events — that's the service layer's job | 🟡 |

---

## Code Quality Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Swallowed exception | `catch (e) {}`, `catch (e) { return null; }` — error lost silently | 🟠 |
| Unhandled promise rejection | `async fn()` called without `await`, no `.catch()` on promise chain | 🟠 |
| Missing error context | `throw new Error('failed')` — no information about what failed or why | 🟡 |
| Magic numbers/strings | `if (status === 3)`, `setTimeout(fn, 86400000)` — unexplained literals | 🟡 |
| Commented-out code | `// const old = doOldThing()` left in codebase | 🟡 |
| Dead code | Exported function never imported, unreachable `if` branch | 🟡 |
| Inconsistent naming | `getUser`, `fetchOrder`, `retrieveProduct` — no naming convention | 🟡 |
| Missing type safety | `any` in TypeScript, missing Python type hints, missing C# nullable | 🟠 |
| Boolean trap | `setUser(user, true, false, true)` — what do the booleans mean? | 🟡 |
| Deep nesting | 4+ levels of `if/for/try` nesting — use early returns, extract functions | 🟡 |

---

## Best Practice Enforcement Checklist

When generating code, verify each before output:

- [ ] Each class/function has one clear, named responsibility
- [ ] All dependencies injected through constructor or parameter — no `new` inside business logic
- [ ] All errors caught and enriched with context before re-throwing or logging
- [ ] No framework types (Request, Response, ORM decorators) in domain/service layer
- [ ] Value Objects used for domain primitives (Money, Email, UserId)
- [ ] Repository interfaces defined in domain; implementations in infrastructure
- [ ] No layer skipping — controller → service → repository, no shortcuts
