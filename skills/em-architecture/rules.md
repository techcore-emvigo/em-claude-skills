# em-architecture — Rules Reference

Non-negotiable rules enforced during every architecture review, API design, system
design, and data layer review task.

Prefix: **AR** = Architecture | **AP** = API Design | **GQ** = GraphQL |
**GR** = gRPC | **WH** = Webhooks | **MQ** = Message Brokers & Queues |
**DB** = Database | **TP** = Third-Party Patterns

---

## 1. Architecture & Layering

| Rule | Description | Severity |
|---|---|---|
| AR-01 | Dependency arrows always point inward — domain/core layer has zero framework or infrastructure imports | 🔴 |
| AR-02 | No circular dependencies between layers or modules | 🔴 |
| AR-03 | Business logic in service/application layer only — never in controllers, routes, or repositories | 🔴 |
| AR-04 | Repositories return domain objects — never raw DB rows, ORM entities, or DB-specific types | 🟠 |
| AR-05 | Controllers are thin — validate input, delegate to service, return response. No logic | 🟠 |
| AR-06 | No cross-service DB access — service A never queries service B's database tables directly | 🔴 |
| AR-07 | No shared mutable state across microservice boundaries | 🔴 |
| AR-08 | Every integration with a third-party SDK or external service is behind an interface (wrapper pattern) | 🔴 |
| AR-09 | One module/bounded context per domain feature — no god modules | 🟠 |
| AR-10 | Monolith first for teams ≤ 5 — do not start with microservices without a platform team | 🟠 |
| AR-11 | Microservice extraction via Strangler Fig pattern — never big-bang rewrites | 🟠 |
| AR-12 | SOLID — SRP, OCP, LSP, ISP, DIP applied at class, module, and service level | 🟠 |

---

## 2. SOLID Principles

| Rule | Description | Severity |
|---|---|---|
| SL-01 | **SRP**: Every class has one reason to change — god classes split by responsibility | 🟠 |
| SL-02 | **OCP**: New behaviour added via abstraction — no growing `if/else if` or `switch` on type | 🟠 |
| SL-03 | **LSP**: Subclasses fully substitutable — no `throw NotImplementedException` in overrides | 🟠 |
| SL-04 | **ISP**: Interfaces are small and focused — clients never depend on methods they don't use | 🟡 |
| SL-05 | **DIP**: High-level modules depend on interfaces — `new ConcreteRepo()` never inside a service | 🔴 |
| SL-06 | Composition over inheritance — max 2 levels of inheritance in application code | 🟠 |
| SL-07 | Value objects used for domain primitives — `Money`, `Email`, `UserId` not raw strings/numbers | 🟡 |
| SL-08 | Anemic domain model avoided — business rules live on entities, not scattered across services | 🟠 |

---

## 3. REST API Design

| Rule | Description | Severity |
|---|---|---|
| AP-01 | Resources are plural nouns — `/users`, `/orders`, never `/getUsers`, `/createOrder` | 🟠 |
| AP-02 | URLs lowercase with hyphens — `/product-categories` not `/ProductCategories` | 🟡 |
| AP-03 | HTTP methods match intent — GET (read), POST (create), PUT/PATCH (update), DELETE (remove) | 🟠 |
| AP-04 | Version in URL path — `/v1/users`; breaking changes require new version | 🟠 |
| AP-05 | Max 2 levels of URL nesting — `/users/{id}/orders` not `/users/{id}/orders/{id}/items/{id}` | 🟡 |
| AP-06 | Correct HTTP status codes — 201 (created), 204 (no content), 422 (validation), 409 (conflict) | 🟠 |
| AP-07 | Never always 200 OK — errors return appropriate 4xx/5xx codes | 🟠 |
| AP-08 | Consistent error envelope — `{ error: { code, message, requestId } }` on every error | 🟠 |
| AP-09 | All list endpoints paginated — cursor-based for large datasets, never unbounded | 🔴 |
| AP-10 | `Location` header included on 201 Created responses | 🟡 |
| AP-11 | Sensitive data never in GET URL parameters | 🔴 |
| AP-12 | Response DTOs whitelist fields — raw DB entities never returned directly | 🟠 |
| AP-13 | `gzip`/`brotli` compression on responses > 1KB | 🟡 |

---

## 4. GraphQL

| Rule | Description | Severity |
|---|---|---|
| GQ-01 | Query depth limit configured — `depthLimit(7)` minimum | 🔴 |
| GQ-02 | Query complexity limit configured — cost estimator + max complexity threshold | 🔴 |
| GQ-03 | Introspection disabled or auth-gated in production | 🟠 |
| GQ-04 | Rate limiting per user per operation — stricter on mutations | 🔴 |
| GQ-05 | DataLoader used on every resolver that fetches a related entity — no N+1 | 🔴 |
| GQ-06 | Mutations return payload type with `errors: [UserError!]!` — not entity directly | 🟡 |
| GQ-07 | Field-level authorization in resolvers — never rely on gateway check alone | 🔴 |
| GQ-08 | All types and fields documented with `"""descriptions"""` | 🟡 |
| GQ-09 | Persisted queries in production — prevents arbitrary query execution | 🟠 |
| GQ-10 | Schema breaking changes detected in CI — snapshot test or `graphql-inspector` | 🟠 |

---

## 5. gRPC

| Rule | Description | Severity |
|---|---|---|
| GR-01 | TLS on all gRPC connections — never plain HTTP/2 (`insecure`) in production | 🔴 |
| GR-02 | Auth validated in a server interceptor — never per-handler | 🟠 |
| GR-03 | Deadline set on every client call — gRPC has no default timeout | 🟠 |
| GR-04 | Deadline propagated downstream — never create `context.Background()` for sub-calls | 🟠 |
| GR-05 | gRPC channel reused — never created per request (connection exhaustion) | 🔴 |
| GR-06 | L7 load balancer in production — L4 (TCP) does not balance gRPC connections | 🟠 |
| GR-07 | Deleted field numbers added to `reserved` — never reused | 🔴 |
| GR-08 | `google.protobuf.Timestamp` for all date/time fields — never string or int64 epoch | 🟡 |
| GR-09 | First enum value always `UNSPECIFIED = 0` | 🟡 |
| GR-10 | `buf breaking` check in CI — detects breaking proto changes before deployment | 🟠 |

---

## 6. Webhooks

| Rule | Description | Severity |
|---|---|---|
| WH-01 | Signature verified against raw body using `timingSafeEqual` before any processing | 🔴 |
| WH-02 | `200 OK` returned within 5 seconds — payload enqueued for async processing | 🔴 |
| WH-03 | Idempotency check on `eventId` before processing — duplicates silently ignored | 🔴 |
| WH-04 | DLQ configured — permanently failed events never silently discarded | 🔴 |
| WH-05 | Unknown event types return `200` and are logged — never throw an error | 🟠 |
| WH-06 | All received webhook payloads logged with `eventId` and `eventType` | 🟠 |
| WH-07 | Outbound webhooks signed with HMAC-SHA256 + timestamp — replay prevention | 🔴 |
| WH-08 | Outbound delivery retried with exponential backoff — permanent failure after max retries | 🟠 |

---

## 7. Message Brokers & Queues

| Rule | Description | Severity |
|---|---|---|
| MQ-01 | Every consumer is idempotent — dedup on `messageId` before processing | 🔴 |
| MQ-02 | DLQ configured on every consumer — failed messages never lost | 🔴 |
| MQ-03 | Every message includes: `messageId`, `correlationId`, `eventType`, `version`, `timestamp` | 🟠 |
| MQ-04 | Schema validated on every consumer entry point before processing | 🟠 |
| MQ-05 | Consumer lag monitored and alerted — queue depth growth triggers investigation | 🟠 |
| MQ-06 | Kafka: `acks=all`, `enable.idempotence=true`, manual offset commit after success | 🔴 |
| MQ-07 | Kafka: replication factor ≥ 3 in production | 🔴 |
| MQ-08 | RabbitMQ: `durable: true` exchanges and queues, `deliveryMode: 2`, DLX configured | 🔴 |
| MQ-09 | SQS: DLQ configured, `WaitTimeSeconds: 20` (long polling), visibility timeout > processing time | 🔴 |
| MQ-10 | Poison pill handled — max receive count set; malformed messages routed to DLQ | 🟠 |
| MQ-11 | `correlationId`/`traceparent` propagated in all message headers for distributed tracing | 🟠 |

---

## 8. Database Design

| Rule | Description | Severity |
|---|---|---|
| DB-01 | Every table has a surrogate primary key — `BIGSERIAL` or `UUID` | 🟠 |
| DB-02 | All FK columns indexed | 🔴 |
| DB-03 | All columns used in WHERE, ORDER BY, GROUP BY indexed | 🔴 |
| DB-04 | Never `SELECT *` — always name required columns | 🟠 |
| DB-05 | Parameterised queries always — no string interpolation in SQL | 🔴 |
| DB-06 | All list queries have `LIMIT` — never unbounded result sets | 🟠 |
| DB-07 | Keyset/cursor pagination for large tables — `OFFSET N` forbidden beyond 1000 rows | 🟠 |
| DB-08 | `DECIMAL` for money — never `FLOAT` or `DOUBLE` | 🔴 |
| DB-09 | `TIMESTAMPTZ` (PostgreSQL) / UTC-enforced for all timestamp columns | 🟡 |
| DB-10 | Mandatory audit columns on every entity: `created_at`, `created_by`, `updated_at`, `updated_by`, `is_deleted` | 🟠 |
| DB-11 | Table names: `snake_case` plural. Column names: `snake_case`. Consistent across entire schema | 🟠 |
| DB-12 | Every migration has a rollback (`down`) counterpart and is idempotent | 🟠 |
| DB-13 | Redis: TTL on every key — never `SET key value` without `EX` | 🟠 |
| DB-14 | Redis: `SCAN` not `KEYS` in any production code | 🔴 |
| DB-15 | MongoDB: fields in camelCase — consistent throughout all collections | 🟡 |
| DB-16 | Firestore: security rules deny-by-default; `request.auth != null` on every rule | 🔴 |

---

## 9. Third-Party Integration Patterns

| Rule | Description | Severity |
|---|---|---|
| TP-01 | Every third-party integration behind an interface — no direct SDK in business logic or UI | 🔴 |
| TP-02 | Domain errors map to integration errors — Stripe errors never leak to calling service | 🟠 |
| TP-03 | Circuit breaker on every critical third-party dependency | 🔴 |
| TP-04 | Timeout on every outbound HTTP/gRPC call — no default timeout assumed | 🟠 |
| TP-05 | Retry with exponential backoff + jitter on transient errors (5xx, network) | 🟠 |
| TP-06 | No retry on 4xx errors — client errors are not transient | 🟠 |
| TP-07 | Fallback behaviour defined — app degrades gracefully when provider is down | 🟠 |
| TP-08 | Rate limit (429) handled — `Retry-After` header respected | 🟠 |
| TP-09 | Idempotency key on every mutating third-party call | 🔴 |
| TP-10 | Third-party response schema validated — never used directly without shape check | 🟠 |
| TP-11 | Audit log written async (off critical path) for every third-party request/response | 🔴 |
| TP-12 | Request status tracked in DB (PENDING / SUCCESS / FAILED) for every integration call | 🟠 |

---

## 10. Instant Escalation — 🔴 Flag Immediately

| # | Violation |
|---|---|
| ESC-01 | Domain layer imports framework, ORM, or HTTP types directly |
| ESC-02 | Circular dependency between architectural layers |
| ESC-03 | Service A directly queries Service B's database tables |
| ESC-04 | Event consumer with no idempotency check — duplicates corrupt data |
| ESC-05 | No DLQ on any async consumer — failed messages silently lost |
| ESC-06 | Webhook handler missing HMAC signature verification |
| ESC-07 | GraphQL with no depth or complexity limits — public DoS risk |
| ESC-08 | gRPC client call with no deadline — can hang indefinitely |
| ESC-09 | gRPC channel created per request — connection pool exhaustion |
| ESC-10 | Third-party SDK called directly from business logic — no wrapper interface |
| ESC-11 | Redis `KEYS *` in production code — blocks event loop |
| ESC-12 | Firestore security rules allow unrestricted read/write |
| ESC-13 | No FK index on a join column in a high-traffic query |
| ESC-14 | Keyset-less pagination (`OFFSET 50000`) on a table with millions of rows |
