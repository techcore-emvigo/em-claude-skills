# Microservices Architecture

## What It Is
Microservices decompose an application into small, independently deployable services, each
owning its data and communicating over the network. Each service is built around a business
capability and can be deployed, scaled, and changed independently.

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│  Users   │    │  Orders  │    │ Payments │
│ Service  │    │ Service  │    │ Service  │
│  + DB    │    │  + DB    │    │  + DB    │
└────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │
     └───────────────┼───────────────┘
              Message Bus / API Gateway
```

---

## Core Principles

### 1. Service Owns Its Data
Each service has its own database — no service ever queries another service's DB directly.
Communication is always through the service's API or events.

```
// ❌ CRITICAL violation: OrdersService queries UsersService DB
SELECT * FROM users_db.users WHERE id = ?   -- cross-service DB access

// ✅ Correct: OrdersService calls UsersService API
GET /users/{id}  -- or subscribe to UserCreated event and cache what it needs
```

### 2. Communicate via API or Events
- **Synchronous**: REST or gRPC for request/response (when caller needs an immediate answer)
- **Asynchronous**: Message bus (Kafka, RabbitMQ, SQS) for events and commands that don't need an immediate response
- Prefer async where possible — it decouples services and improves resilience

### 3. Design Around Business Capabilities (DDD Bounded Contexts)
- Each service = one bounded context
- Bounded contexts have their own ubiquitous language — `Order` in OrdersService may be different from `Order` in ShippingService
- Don't map to database tables — map to business capabilities

### 4. Build for Failure
Any network call can fail. Services must be resilient:
- **Circuit Breaker**: stop calling a failing service; give it time to recover
- **Retry with backoff**: transient failures are normal
- **Timeout**: every inter-service call must have a timeout
- **Bulkhead**: isolate failures — don't let one failing service crash others
- **Fallback**: degrade gracefully when a dependency is unavailable

---

## Service Design Best Practices

### Service Boundaries
- One team can own and deploy a service independently
- If two services are always deployed together, they may be one service
- If one business change always requires changing multiple services, boundaries are wrong
- Start larger and split — don't start with nano-services

### API Design Between Services
- Version all service APIs: `/v1/orders`
- Backward-compatible changes only in a version (add fields, don't remove)
- Consumer-driven contract tests (Pact) to catch breaking changes
- Define a service's API in OpenAPI/Protobuf and commit it as the contract

### Data Consistency
- **No distributed transactions** — use the Saga pattern instead
- **Saga (Choreography)**: services react to each other's events, no central coordinator
- **Saga (Orchestration)**: a dedicated orchestrator service directs the workflow
- **Eventual consistency** is the norm — design UIs and business flows to tolerate it
- **Idempotency keys** on all state-changing operations — events can be delivered more than once

```typescript
// ✅ Saga choreography: each service reacts to events
// OrdersService publishes:
eventBus.publish('order.created', { orderId, items, userId });

// InventoryService listens and reserves:
eventBus.subscribe('order.created', async (e) => {
  await inventory.reserve(e.items);
  eventBus.publish('inventory.reserved', { orderId: e.orderId });
});

// PaymentsService listens and charges:
eventBus.subscribe('inventory.reserved', async (e) => { /* ... */ });
```

### Service Communication Patterns
- **Synchronous (REST/gRPC)**: use for: user-facing queries, real-time data needs, simple request-response
- **Async (Events/Queue)**: use for: commands with side effects, fan-out notifications, decoupled workflows
- **Never**: synchronous chains of 3+ services for a single user request (latency multiplies, failures cascade)

---

## API Gateway Pattern

```
Client → API Gateway → [UserService, OrderService, PaymentService]
```

API Gateway responsibilities:
- SSL termination
- Authentication (validate JWT, forward user context)
- Rate limiting and throttling
- Request routing
- Response aggregation (BFF pattern)
- Logging and tracing injection

**Never put business logic in the API Gateway** — it becomes a bottleneck and a god service.

---

## Service Discovery & Configuration
- Service registry: Consul, Kubernetes DNS, AWS Cloud Map
- Config: environment variables + secrets manager — never hardcoded config files
- Feature flags: LaunchDarkly or similar — per-service feature control
- Each service reads its own config at startup; restart to pick up changes (or use config push)

---

## Distributed Tracing & Observability
- Propagate `trace_id` across ALL inter-service calls via headers (`traceparent` W3C standard)
- Structured JSON logs with `trace_id`, `service`, `environment` on every line
- Centralized log aggregation (Datadog, Elastic, Loki)
- Metrics per service: RED (Rate, Errors, Duration) minimum
- Health endpoints on every service: `/health` (liveness) + `/ready` (readiness)
- See `references/observability.md` for full details

---

## Testing Strategy

| Test Type | Scope | Tool |
|---|---|---|
| Unit | Single service, mocked dependencies | Jest, Pytest |
| Integration | Service + real DB/cache | Testcontainers |
| Contract | API contract between two services | Pact |
| Component | Full service in isolation, dependencies mocked | Docker Compose |
| E2E | Multiple real services | Dedicated test environment |

Contract tests are **non-negotiable** in microservices — without them, deployments break each other silently.

---

## Common Microservice Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Distributed Monolith | Services coupled at DB or deploy level | Fix boundaries; each service owns its DB |
| Chatty Services | Service A calls B calls C calls D synchronously | Async events; or consolidate into fewer services |
| Shared DB | Two services write to the same tables | Each service owns its tables; sync via events |
| No Circuit Breaker | Cascade failures bring down everything | Add Resilience4j / polly / `cockatiel` |
| Synchronous Saga | Distributed transaction via REST calls | Saga with message bus + compensation logic |
| No Contract Tests | Services break each other on deploy | Add Pact or similar contract testing |
| Premature Decomposition | Split before domain is understood | Start modular monolith; extract when needed |

---

## Microservices Readiness Checklist

Before adopting microservices, confirm:
- [ ] You have a DevOps platform team (Kubernetes/container expertise)
- [ ] You have distributed tracing in place
- [ ] You have a CI/CD pipeline per service
- [ ] Your team is large enough that a monolith causes coordination pain
- [ ] Domain boundaries are well understood
- [ ] You have a service mesh or API gateway
- [ ] Teams are organized around business capabilities (Conway's Law)

**If most of these are "no" → start with a Modular Monolith instead.**
