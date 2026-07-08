# Monolithic Architecture

## What It Is
A monolith is a single deployable unit containing all application functionality. It is NOT
inherently bad — for many teams and products it is the right choice. The goal is a
**well-structured monolith**, not a "big ball of mud."

```
┌──────────────────────────────────────────────────┐
│                Single Process / Deployment        │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
│  │  Users   │  │  Orders  │  │   Payments     │  │
│  │  Module  │  │  Module  │  │   Module       │  │
│  └──────────┘  └──────────┘  └────────────────┘  │
│                                                   │
│  ┌──────────────────────────────────────────────┐ │
│  │          Shared Database                     │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

---

## Types of Monolith

### 1. Single-Process Monolith (Classic)
All code in one process, one deployment, one database. Simple to develop, test, and deploy.

### 2. Modular Monolith (Recommended Evolution)
Single deployment but internally divided into well-bounded modules with explicit APIs between
them. **This is the best starting point for most products.**

```
Monolith Process
├── UsersModule     ← owns User DB tables, exposes UserService interface
├── OrdersModule    ← owns Order DB tables, calls UserService (not UserRepo!)
├── PaymentsModule  ← owns Payment DB tables, publishes internal events
└── Shared/
    └── EventBus    ← in-process event bus for loose coupling between modules
```

### 3. Distributed Monolith (Anti-Pattern — Avoid)
Multiple deployed services that are tightly coupled to each other (shared DB, synchronous
chains, coupled deploys). Worst of both worlds: distributed complexity without the benefits
of microservices.

---

## Modular Monolith Best Practices

### Module Boundaries
- Each module owns its data — no other module queries its tables directly
- Modules communicate through explicit interfaces (service classes) or internal events
- Enforce boundaries with linting rules or package structure (e.g., separate npm packages, NestJS modules)
- Circular module dependencies are forbidden

```typescript
// ❌ Bad: OrdersModule bypasses UserModule's boundary
import { UserRepository } from '../users/user.repository'; // direct DB access

// ✅ Good: OrdersModule uses the module's public interface
import { UserService } from '../users/user.service'; // public contract only
```

### Database Strategy
- One schema, but tables are owned by a module — naming convention enforces this: `users_*`, `orders_*`, `payments_*`
- Cross-module joins are fine in a monolith — but avoid them at the service layer; join at DB level
- Track which tables belong to which module — future extraction to microservices becomes easier

### Internal Event Bus
Use an in-process event bus for decoupling modules without HTTP overhead:
```typescript
// PaymentsModule emits — doesn't know who listens
eventBus.publish(new PaymentCompletedEvent({ orderId, amount }));

// OrdersModule reacts independently
eventBus.subscribe(PaymentCompletedEvent, async (event) => {
  await this.orderService.markAsPaid(event.orderId);
});
```

### Anti-Corruption Layers Between Modules
When one module's model leaks into another, use a translation layer:
```typescript
// OrdersModule doesn't use UserEntity — maps to its own representation
class OrderUserSnapshot {
  static fromUser(user: User): OrderUserSnapshot {
    return new OrderUserSnapshot(user.id, user.fullName, user.email);
  }
}
```

---

## When Monolith is the Right Choice

✅ **Choose monolith when:**
- Team is 1–10 developers
- Product is early-stage — domain boundaries not fully understood yet
- Deployment simplicity is important (one build, one deploy, one logs source)
- You want a fast feedback loop for feature development
- The domain is simple or medium complexity

⚠️ **Signs you may need to split (extract, not rewrite):**
- A single module's deployment blocks all other modules for unrelated changes
- A specific part needs to scale independently (e.g., video processing, ML inference)
- Teams stepping on each other constantly in the same codebase
- A module has completely different SLAs or tech requirements

**Migrate to microservices one service at a time (Strangler Fig pattern) — never "big bang" rewrites.**

---

## Monolith Quality Checklist

| Check | Good Sign | Bad Sign |
|---|---|---|
| Module coupling | Modules communicate via interfaces/events | Modules import each other's repos/entities |
| Database access | Each module owns its tables | Free-for-all table access across modules |
| Deployment | One command deploys everything | Multi-step, fragile deploy process |
| Test isolation | Each module tested independently | Integration tests require the whole app |
| Startup time | < 10 seconds | Slow startup blocks developer productivity |
| Shared state | Minimal global state | Many singletons with mutable shared state |

---

## Transitioning Monolith → Microservices (Strangler Fig)

1. **Identify the seam** — find a module with clear boundaries and independent scaling needs
2. **Create a façade** — in-process interface in the monolith that matches the future service API
3. **Extract the module** — move code to a separate service, keep the façade
4. **Route traffic** — façade calls the new service instead of internal code
5. **Remove the façade** — once the new service is stable, delete the in-process code
6. **Repeat** — one service at a time, not all at once

```
Phase 1: Monolith with façade
Monolith → [OrdersFacade] → internal OrdersModule

Phase 2: Façade routes to new service
Monolith → [OrdersFacade] → HTTP → OrdersService (new)

Phase 3: Façade removed
Monolith → HTTP → OrdersService (stable)
```
