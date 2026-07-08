# Serverless Architecture

## What It Is
Serverless means the cloud provider manages the infrastructure — you deploy functions or
containers that run on-demand, scale automatically, and charge per execution. "Serverless"
doesn't mean no servers; it means you don't manage them.

```
Client → API Gateway → Lambda / Cloud Function → DB / Queue / Storage
                                                ↑
                                   (scales 0 → N automatically)
```

**Main platforms:**
- AWS: Lambda, API Gateway, EventBridge, SQS, Step Functions
- GCP: Cloud Functions, Cloud Run, Pub/Sub, Workflows
- Azure: Functions, Event Grid, Service Bus, Durable Functions
- Framework-agnostic: Vercel Functions, Netlify Functions, Cloudflare Workers

---

## Function Design Principles

### Single Responsibility per Function
- One function = one clear purpose: `processPayment`, `sendWelcomeEmail`, `resizeImage`
- If a function handles multiple event types with branching logic, split it
- Keep functions under 150–200 lines; extract shared logic to shared libraries/layers

### Stateless by Design
- Functions must not rely on local disk state between invocations (filesystem is ephemeral)
- Store state in: DynamoDB, RDS, Redis, S3, or a message queue
- In-memory caching is OK for the lifetime of one warm instance — never rely on it being there

### Cold Start Awareness
- Cold starts add latency (100ms–2s depending on runtime and memory)
- Minimize cold starts: use provisioned concurrency for latency-critical paths
- Prefer lightweight runtimes: Node.js and Python start faster than Java/.NET
- Keep function packages small: tree-shake, use Lambda Layers for shared deps
- Don't import the entire SDK — import only what you need

```typescript
// ❌ Bad: imports entire AWS SDK
import AWS from 'aws-sdk';

// ✅ Good: import only what's needed
import { DynamoDBClient, GetItemCommand } from '@aws-sdk/client-dynamodb';
```

### Timeouts
- **Always set a timeout** — the maximum time a function should ever run
- AWS Lambda max: 15 minutes; typical API functions: 3–10 seconds
- Async/background functions: 30 seconds – 5 minutes depending on task
- If a task may exceed the timeout, use Step Functions or a job queue instead
- Never assume a function will run to completion — handle partial execution

---

## Security Best Practices

### Least Privilege IAM
- Each function gets its own IAM role with only the permissions it needs
- Never share one IAM role across all functions
- Never use wildcard resources: `arn:aws:s3:::*` — specify the exact bucket ARN
- No `*` actions — enumerate exactly what each function can do

```json
// ❌ Bad: wildcard permissions
{ "Effect": "Allow", "Action": "*", "Resource": "*" }

// ✅ Good: least privilege
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem", "dynamodb:PutItem"],
  "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/Orders"
}
```

### Secrets Management
- Never hardcode secrets in function code or environment variables in IaC files
- Use AWS Secrets Manager / GCP Secret Manager / Azure Key Vault
- Cache secrets in memory within the function lifecycle — don't fetch on every invocation
- Rotate secrets automatically; functions pick up new values on next cold start or re-deploy

### Input Validation
- Validate all inputs at the function entry point — never trust event payload structure
- API Gateway: use request validators or Lambda Authorizers for auth
- Validate event sources — Lambda can be invoked by many triggers; verify the source is expected
- Never pass user input directly to shell commands, DB queries, or `eval`

### Network Security
- Run functions inside a VPC for DB access — don't expose DB ports publicly
- Use VPC endpoints for AWS services — avoid internet egress for internal calls
- Outbound: restrict with security group egress rules

---

## Performance Patterns

### Connection Pooling Outside the Handler
DB connections are expensive to create. Initialize at module level, not inside the handler:
```typescript
// ✅ Good: connection created once per warm instance, reused across invocations
const db = new Pool({ connectionString: process.env.DB_URL, max: 1 }); // max 1 per Lambda

export const handler = async (event) => {
  const result = await db.query('SELECT * FROM orders WHERE id = $1', [event.orderId]);
  return result.rows[0];
};
```

### Async Fan-Out
Use parallel invocations for independent work:
```typescript
// ❌ Bad: sequential processing of 100 records
for (const record of records) { await processRecord(record); }

// ✅ Good: parallel fan-out (within concurrency limits)
await Promise.all(records.map(r => processRecord(r)));
// Or: send each record to an SQS queue and let Lambda scale consumers
```

### Right-Sizing Memory
- Lambda: more memory = more CPU = faster execution = potentially lower cost
- Profile real workloads with AWS Lambda Power Tuning
- Don't default to 128MB for everything — compute-heavy tasks need more

---

## Orchestration: Step Functions / Durable Functions

For workflows with multiple steps, retries, and branching — use an orchestrator, not chained Lambda calls.

```
// ❌ Bad: chained Lambda calls (fragile, no visibility, no retry)
LambdaA → invokes → LambdaB → invokes → LambdaC

// ✅ Good: Step Functions orchestrates
StepFunction:
  Step1: ProcessOrder (retry 3x on failure)
  Step2: ReserveInventory (parallel with ProcessPayment)
  Step3: SendConfirmation
  OnError: CancelOrder (compensation)
```

**Use Step Functions / Durable Functions when:**
- Workflow has 3+ sequential steps
- Steps need retry logic with backoff
- You need to wait for a human or external event (wait-for-callback)
- You need visibility into workflow progress
- Long-running processes (hours/days) with checkpoints

---

## Event Sources & Triggers

| Trigger | Pattern | Watch Out |
|---|---|---|
| API Gateway (HTTP) | Request/response | Timeout 29s max (APIGW limit) |
| SQS Queue | Async processing | Batch size affects throughput; DLQ required |
| SNS | Fan-out pub/sub | At-least-once; consumers must be idempotent |
| EventBridge | Scheduled / event-driven | Schema registry for event contracts |
| S3 Events | File processing | Only trigger on specific prefixes/suffixes |
| DynamoDB Streams | CDC / event sourcing | Process in order per partition key |
| Kinesis | High-volume streaming | At-least-once; sequence ordering per shard |

---

## Serverless Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Lambda calling Lambda synchronously | Tight coupling, cascading failures, double billing | Use SQS/EventBridge between functions |
| No timeout set | Function hangs, burns cost and concurrency | Set timeout on every function |
| DB connection per invocation | Connection pool exhausted, slow | Initialize connection at module level |
| Monolith Lambda | One function handles all routes | Split by endpoint/responsibility |
| Secrets in environment variables (plaintext) | Secrets visible in console/logs | Use Secrets Manager |
| No DLQ on async triggers | Failed events silently lost | Add DLQ to SQS, SNS, EventBridge |
| Wildcard IAM roles | Over-permissioned, security risk | Least privilege per function |
| Ignoring cold starts for latency-sensitive APIs | P99 spikes | Provisioned concurrency or warm-up |

---

## Serverless Readiness Checklist

- [ ] Each function has a specific timeout set
- [ ] Each function has a dedicated least-privilege IAM role
- [ ] Secrets fetched from Secrets Manager, not plaintext env vars
- [ ] DB connections initialized outside handler
- [ ] All async triggers have a DLQ configured
- [ ] Input validation on all event payloads
- [ ] Idempotency keys used for state-changing functions
- [ ] Cold start profiled for latency-sensitive paths
- [ ] Step Functions used for multi-step workflows
- [ ] Distributed tracing enabled (AWS X-Ray, GCP Cloud Trace)
