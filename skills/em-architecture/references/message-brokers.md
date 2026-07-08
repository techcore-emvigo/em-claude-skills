# Message Brokers & Queues — Gap Detection

## Universal Async Gaps (All Brokers)

| Gap | What to Look For | Severity |
|---|---|---|
| No idempotency on consumer | Message processed without deduplication check on message ID | 🔴 |
| No DLQ configured | Consumer with no dead-letter queue — failed messages disappear | 🔴 |
| No schema validation | Consumer uses message fields without validating payload shape | 🟠 |
| Missing correlation/trace ID | Messages with no `correlationId` — distributed trace broken | 🟠 |
| No consumer lag alerting | Queue depth or consumer lag not monitored | 🟠 |
| Poison pill not handled | Malformed message retried forever, blocking the queue | 🟠 |
| Message too large (> broker limit) | Large payloads embedded in message vs referenced via storage | 🟠 |

## Kafka Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Auto-commit enabled | `enable.auto.commit=true` — offset committed before processing succeeds | 🔴 |
| Replication factor < 3 | `replication.factor=1` or `2` — no fault tolerance | 🔴 |
| `acks=1` on critical topic | Producer not waiting for all replicas — messages can be lost | 🟠 |
| No idempotent producer | `enable.idempotence=false` — duplicate messages on retry | 🟠 |
| Random partition key | No partition key or random key — ordering lost for related events | 🟡 |
| No schema registry | Raw JSON without schema enforcement on high-scale topics | 🟠 |
| No Dead Letter Topic | Failed messages with no DLT configured | 🔴 |

## RabbitMQ Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Non-durable queue/exchange | `durable: false` — all messages lost on broker restart | 🔴 |
| Non-persistent messages | `deliveryMode: 1` (transient) on important messages | 🔴 |
| No prefetch / QoS | `channel.prefetch()` not set — slow consumer flooded with all messages | 🟠 |
| No DLX configured | Queue with no `x-dead-letter-exchange` argument | 🔴 |
| `nack` with `requeue: true` always | Permanent failures requeued forever — infinite retry loop | 🟠 |
| Channel shared across threads | Single `channel` used from multiple goroutines/threads | 🔴 |

## SQS / SNS / EventBridge Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No DLQ on SQS queue | `redrive_policy` not configured — failed messages disappear after max receives | 🔴 |
| Short polling | `WaitTimeSeconds: 0` — wasteful and increases cost | 🟡 |
| Visibility timeout too short | Timeout shorter than max processing time — causes double processing | 🔴 |
| No message retention configured | Default 4-day retention may not meet recovery SLA | 🟡 |
| SQS Standard for ordered work | Using Standard queue where ordering or deduplication is required | 🟠 |
| Lambda trigger with no error handling | Lambda SQS trigger with no `try/catch` — message deleted even on failure | 🔴 |

## Generation Checklist
- [ ] Every consumer: idempotency check before processing (Redis `SET NX`)
- [ ] Every consumer: DLQ configured; DLQ depth alert set
- [ ] Every message: `messageId`, `correlationId`, `eventType`, `timestamp` in envelope
- [ ] Kafka: `acks=all`, `enable.idempotence=true`, manual offset commit after success
- [ ] RabbitMQ: `durable: true` queues and exchanges, `deliveryMode: 2`, prefetch set, DLX configured
- [ ] SQS: long polling (`WaitTimeSeconds: 20`), visibility timeout > processing time, DLQ configured
- [ ] Schema validated on every consumer entry point
