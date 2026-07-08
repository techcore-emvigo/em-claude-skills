# Event-Driven Architecture (EDA)

## What It Is
Event-Driven Architecture decouples producers and consumers through events — facts that
something happened. Producers publish events without knowing who (if anyone) is listening.
Consumers react to events independently and asynchronously.

```
[Producer]  →  event: "order.placed"  →  [Message Bus]
                                               ├──→ InventoryService (reserves stock)
                                               ├──→ NotificationService (sends email)
                                               └──→ AnalyticsService (logs event)
```

---

## Core Concepts

### Event vs Command vs Query
- **Event**: "Something happened" — past tense, no expectation of response (`OrderPlaced`, `PaymentFailed`)
- **Command**: "Do this" — imperative, directed at a specific handler (`ProcessPayment`, `SendEmail`)
- **Query**: "Tell me this" — request for data, typically synchronous
- Events are immutable facts. Never edit or delete published events.

### Event Types
- **Domain Events**: business facts (`UserRegistered`, `OrderShipped`) — the core of EDA
- **Integration Events**: cross-service domain events published to a message bus
- **Change Data Capture (CDC)**: DB-level events (Debezium) used for data sync and audit

---

## Event Design Best Practices

### Event Schema
```json
{
  "eventId":   "evt_abc123",          // unique, for deduplication
  "eventType": "order.placed",        // namespaced type
  "version":   "1.0",                 // for schema evolution
  "timestamp": "2024-01-15T10:30:00Z",
  "source":    "orders-service",
  "correlationId": "req_xyz",         // traces the user request across services
  "payload": {
    "orderId": "ord_456",
    "userId":  "usr_789",
    "items":   [...]
  }
}
```

### Rules for Good Events
- Events are **immutable** — never update a published event
- Events are **self-contained** — include all data consumers need, or enough to fetch it
- Events use **past tense** naming: `OrderPlaced` not `PlaceOrder`
- Include `eventId` for **idempotency** — consumers deduplicate on this
- Include `correlationId` for **distributed tracing** across the event chain
- Version events — add `version` field from day one; plan for schema evolution

### Schema Evolution
- **Backward compatible** (safe): add new optional fields
- **Breaking** (requires versioning): rename, remove, or change type of fields
- Use schema registry (Confluent Schema Registry for Kafka, AWS Glue) to enforce contracts
- Consumers should ignore unknown fields (tolerant reader pattern)

---

## Message Bus & Broker Selection

| Broker | Use Case |
|---|---|
| **Apache Kafka** | High-throughput event streaming, event log, replay, CDC |
| **RabbitMQ** | Task queues, pub/sub, complex routing, lower throughput |
| **AWS SQS/SNS** | Cloud-native queue/pub-sub, serverless integrations |
| **Google Pub/Sub** | GCP-native, at-least-once delivery, push/pull |
| **Redis Streams** | Lightweight event streaming within a single stack |

**Kafka** is the default choice for high-volume EDA at scale. For simpler use cases, SQS/SNS
or RabbitMQ are easier to operate.

---

## Consumer Best Practices

### Idempotency (Non-Negotiable)
At-least-once delivery means duplicate events WILL arrive. Every consumer MUST be idempotent.

```typescript
// ✅ Good: idempotent consumer with deduplication
async handleOrderPlaced(event: OrderPlacedEvent) {
  const alreadyProcessed = await this.processedEvents.exists(event.eventId);
  if (alreadyProcessed) {
    logger.info('Duplicate event ignored', { eventId: event.eventId });
    return;
  }
  await this.inventory.reserve(event.payload.items);
  await this.processedEvents.mark(event.eventId, { ttl: '24h' });
}
```

### Error Handling & Dead Letter Queues
- Transient failures (DB down, timeout): retry with exponential backoff
- Permanent failures (bad data, logic error): send to Dead Letter Queue (DLQ)
- Monitor DLQ depth — messages there need investigation, not silent discard
- Never silently swallow consumer errors

```typescript
// ✅ Good: retry + DLQ pattern
async processWithRetry(event: Event, attempt = 1) {
  try {
    await this.handler.handle(event);
  } catch (err) {
    if (attempt < MAX_RETRIES && isTransient(err)) {
      await sleep(backoff(attempt));
      return this.processWithRetry(event, attempt + 1);
    }
    await this.dlq.send(event, { error: err.message, attempt });
    logger.error('Event sent to DLQ', { eventId: event.eventId, err });
  }
}
```

### Consumer Groups & Scaling
- Use consumer groups (Kafka) or competing consumers (SQS) for horizontal scaling
- Partition by a stable key (`userId`, `orderId`) to ensure ordering within a key
- One consumer group per use case — different consumers of the same event get their own group

---

## Saga Pattern (Distributed Transactions via Events)

Sagas replace distributed transactions in EDA systems.

### Choreography Saga
Services react to each other's events — no central coordinator:
```
OrderPlaced → InventoryService reserves stock → InventoryReserved
                                              → PaymentService charges → PaymentSucceeded
                                                                       → ShippingService dispatches
```
- Simple to implement
- Hard to visualise/debug the overall flow
- Use for simple linear workflows

### Orchestration Saga
A dedicated Saga orchestrator issues commands and reacts to results:
```
SagaOrchestrator:
  1. → command: ReserveInventory → InventoryService
  2. ← event: InventoryReserved
  3. → command: ProcessPayment → PaymentService
  4. ← event: PaymentFailed → compensate: ReleaseInventory
```
- Easier to track state and debug
- Central point of failure/complexity
- Use for complex workflows with many steps or branching logic

### Compensation
If a step fails, run compensating transactions for all completed steps:
```
OrderPlaced → InventoryReserved → PaymentFailed
                                       ↓
                        COMPENSATE: ReleaseInventory → OrderCancelled
```
Compensation must be idempotent and always succeed (or be retried until it does).

---

## Event Sourcing (Advanced EDA)

Instead of storing current state, store the sequence of events that led to it:
```
events:
  OrderCreated   { items: [...] }
  ItemAdded      { sku: "ABC" }
  PaymentTaken   { amount: 99.99 }
  OrderShipped   { trackingId: "TRK123" }

current state = replay of all events
```

**Use when:**
- Full audit trail is a business requirement
- Temporal queries matter ("what was the state on Jan 5th?")
- Debugging or replaying workflows is important

**Don't use when:**
- Simple CRUD — the overhead is not worth it
- Team is unfamiliar — the learning curve is significant
- Querying current state is complex enough to require many projections

---

## EDA Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Events without `eventId` | Can't deduplicate — double-processing | Add `eventId` to every event |
| Non-idempotent consumers | Duplicates corrupt data | Add dedup check before processing |
| Synchronous reply over event | Introduces coupling | Use request/reply pattern or REST |
| Fat events (full entity snapshot) | Breaking changes on any field change | Use lean events + fetch-on-need |
| No DLQ | Failures silently disappear | Add DLQ + alerting on DLQ depth |
| Event spaghetti | Impossible to trace a workflow | Add correlationId + distributed tracing |
| Shared event schemas across all services | One change breaks everyone | Own your event contracts; version explicitly |

---

## EDA Observability Checklist
- [ ] All events include `eventId`, `correlationId`, `timestamp`, `version`
- [ ] DLQ configured and monitored for all consumers
- [ ] Consumer lag monitored (Kafka: consumer group lag; SQS: queue depth)
- [ ] Idempotency implemented on every consumer
- [ ] Distributed tracing propagated via `correlationId` / `traceparent`
- [ ] Schema registry in use for high-volume event streams
- [ ] Saga state persisted — orchestrator can recover after crash
