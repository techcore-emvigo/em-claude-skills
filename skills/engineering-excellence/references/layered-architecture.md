# Layered / N-Tier Architecture Patterns

## What It Is
Layered architecture organises code into horizontal layers, each with a distinct responsibility.
Each layer depends only on the layer directly below it. This is the most common pattern for
web applications and APIs.

---

## Classic 3-Tier (Presentation → Business → Data)

```
┌─────────────────────────────────────────┐
│  Presentation Layer (API / Controllers) │  ← Handles HTTP, input/output, auth
├─────────────────────────────────────────┤
│  Business Logic Layer (Services)        │  ← Domain rules, use cases, workflows
├─────────────────────────────────────────┤
│  Data Access Layer (Repositories / ORM) │  ← DB queries, persistence, caching
└─────────────────────────────────────────┘
```

**Dependency rule:** Each layer ONLY imports from the layer directly below. Presentation never
imports Data Access. Data Access never imports Business Logic.

---

## Clean / Onion / Hexagonal Architecture (Advanced Layering)

These are variations of layered architecture with a strict inward-pointing dependency rule:

```
         ┌──────────────────────────────┐
         │   Infrastructure / Adapters  │  ← DB, HTTP clients, queues, email
         │  ┌────────────────────────┐  │
         │  │  Application / Use Cases│  │  ← Orchestrates domain objects
         │  │  ┌──────────────────┐  │  │
         │  │  │   Domain / Core   │  │  │  ← Entities, Value Objects, Domain Events
         │  │  │  (no deps at all) │  │  │
         │  │  └──────────────────┘  │  │
         │  └────────────────────────┘  │
         └──────────────────────────────┘
```

**The one rule: dependency arrows always point inward. Domain has zero external dependencies.**

### Layer Responsibilities

| Layer | Responsibility | May Import |
|---|---|---|
| **Domain** | Entities, Value Objects, domain rules, domain events | Nothing external |
| **Application** | Use cases, commands, queries, orchestration | Domain only |
| **Infrastructure** | DB, HTTP, queues, email, file storage | Application + Domain interfaces |
| **Presentation** | Controllers, GraphQL resolvers, CLI | Application only |

---

## Layer-by-Layer Best Practices

### Presentation Layer (API / Controllers)
- **Thin controllers only** — no business logic, no SQL
- Validate and deserialize input (DTOs), delegate to services, serialize output
- Map HTTP status codes correctly — don't always return 200
- Handle authentication/authorization here (guards, middleware) — not in services
- Return consistent error envelopes

```typescript
// ✅ Good: thin controller
@Post('/orders')
@UseGuards(AuthGuard)
async createOrder(@Body() dto: CreateOrderDto, @CurrentUser() user: User) {
  const order = await this.orderService.create(dto, user.id); // delegate
  return new OrderResponse(order);                             // transform
}
```

### Business / Application Layer (Services / Use Cases)
- Contains all business rules and application workflows
- Orchestrates domain objects and repositories
- Handles transactions — begin/commit/rollback at this layer
- Throws domain exceptions (e.g., `OrderNotFoundException`) — presentation maps them to HTTP
- Has no knowledge of HTTP, DB schemas, or framework types
- One class per use case (Command Handler pattern) keeps things focused

```typescript
// ✅ Good: service owns workflow, delegates persistence
class OrderService {
  async create(dto: CreateOrderDto, userId: string): Promise<Order> {
    const user = await this.userRepo.findById(userId);
    if (!user) throw new UserNotFoundException(userId);
    const inventory = await this.inventoryRepo.findBySkus(dto.items.map(i => i.sku));
    const order = Order.create(user, dto.items, inventory); // domain logic in entity
    await this.orderRepo.save(order);
    this.eventBus.publish(new OrderCreatedEvent(order));
    return order;
  }
}
```

### Data Access Layer (Repositories)
- One repository per aggregate root (not per table)
- Returns domain objects — not raw DB rows
- Hides DB technology completely — callers never know if it's PostgreSQL, MongoDB, or Redis
- No business logic in repositories — only CRUD + query methods
- Named by domain concept: `findActiveOrdersByUser()` not `selectFromOrdersWhereStatus()`

```typescript
// ✅ Good: repository interface in domain, implementation in infrastructure
// Domain layer:
interface IOrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findActiveByUser(userId: UserId): Promise<Order[]>;
  save(order: Order): Promise<void>;
}

// Infrastructure layer:
class PostgresOrderRepository implements IOrderRepository { ... }
```

### Domain Layer (Entities & Value Objects)
- **Entities**: have identity (ID), mutable state, lifecycle (e.g., `Order`, `User`)
- **Value Objects**: no identity, immutable, defined by value (e.g., `Money`, `Email`, `Address`)
- **Domain Events**: record something that happened (`OrderPlaced`, `PaymentFailed`)
- Domain objects enforce their own invariants — never reach an invalid state
- No framework imports, no DB imports, no HTTP imports — pure business logic only

```typescript
// ✅ Good: entity enforces its own rules
class Order {
  private constructor(private _status: OrderStatus, private _items: OrderItem[]) {}

  static create(user: User, items: OrderItem[]): Order {
    if (items.length === 0) throw new DomainError('Order must have at least one item');
    return new Order(OrderStatus.PENDING, items);
  }

  confirm() {
    if (this._status !== OrderStatus.PENDING) throw new DomainError('Only pending orders can be confirmed');
    this._status = OrderStatus.CONFIRMED;
    this.addEvent(new OrderConfirmedEvent(this));
  }
}
```

---

## Common Layered Architecture Violations

| Violation | Example | Fix |
|---|---|---|
| Fat Controller | SQL query in a controller | Move to service/repository |
| Anemic Service | Service just calls repo with no logic | Move logic from caller into service |
| Domain imports ORM | `Order` imports `@Entity()` from TypeORM | Separate persistence model from domain entity |
| Repo returns raw rows | `findUser()` returns `{ user_id, first_name }` | Map to domain `User` object |
| Cross-layer skip | Controller imports repository directly | Route through service layer |
| Business logic in DB | Complex stored procedures | Move to application layer |
| Circular layer dependency | Service imports a controller type | Restructure — always one direction |

---

## Folder Structure (NestJS / Node Example)

```
src/
├── presentation/
│   ├── controllers/
│   ├── dtos/
│   └── guards/
├── application/
│   ├── commands/        (write use cases)
│   ├── queries/         (read use cases)
│   └── services/
├── domain/
│   ├── entities/
│   ├── value-objects/
│   ├── events/
│   └── repositories/    (interfaces only)
└── infrastructure/
    ├── persistence/     (repository implementations)
    ├── messaging/
    └── http-clients/
```

---

## When to Use Layered Architecture

✅ Use when:
- Standard web API or web application
- Team is familiar with the pattern
- Domain complexity is moderate to high (clear business rules)
- You want testability with clear separation of concerns

⚠️ Reconsider if:
- The application is genuinely simple CRUD — layering adds overhead without benefit
- You need extreme horizontal scaling per concern → Microservices
- You have event-heavy, reactive workflows → Event-Driven
