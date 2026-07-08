# Third-Party Patterns — Resilience & Validation

Covers: request status tracking in DB, circuit breaker, retry with backoff,
rate limit handling, async/parallel bulk processing, schema validation.

---

## 4. Request Status Tracking in Database

Every outbound integration call must have a status record. This enables replay,
reconciliation, debugging of partial failures, and client-facing transparency.

```typescript
// ✅ Good — IntegrationRequest entity tracks every outbound call
// entities/integration-request.entity.ts
@Entity('integration_requests')
export class IntegrationRequest {
  @PrimaryGeneratedColumn('uuid') id:              string;
  @Column() provider:              string;  // 'stripe' | 'sendgrid' | 'twilio'
  @Column() operation:             string;  // 'charge' | 'send_email' | 'send_sms'
  @Column() referenceId:           string;  // our internal ID (orderId, userId, etc.)
  @Column() status:                RequestStatus;
  @Column('jsonb') sanitisedRequest:  object;  // payload with secrets/PII removed
  @Column('jsonb', { nullable: true }) response: object | null;
  @Column({ nullable: true }) externalId: string | null; // provider's transaction ID
  @Column({ default: 1 })   attemptCount: number;
  @Column({ nullable: true }) errorMessage: string | null;
  @Column({ nullable: true }) resolvedAt:   Date | null;
  @CreateDateColumn() createdAt: Date;
  @UpdateDateColumn() updatedAt: Date;
}

enum RequestStatus {
  PENDING  = 'PENDING',
  SUCCESS  = 'SUCCESS',
  FAILED   = 'FAILED',
  RETRYING = 'RETRYING',
  MANUAL_REVIEW = 'MANUAL_REVIEW',
}

// ✅ Repository method: create record → call → update status atomically
// repositories/integration-request.repository.ts
@Injectable()
export class IntegrationRequestRepository {

  async executeTracked<T>(
    provider:     string,
    operation:    string,
    referenceId:  string,
    sanitisedReq: object,
    fn:           (record: IntegrationRequest) => Promise<T>,
  ): Promise<T> {
    const record = await this.repo.save(
      this.repo.create({ provider, operation, referenceId,
        status: RequestStatus.PENDING,
        sanitisedRequest: sanitisedReq,
        attemptCount: 1 })
    );

    try {
      const result = await fn(record);

      await this.repo.update(record.id, {
        status:     RequestStatus.SUCCESS,
        resolvedAt: new Date(),
        externalId: (result as any)?.id ?? null,
      });
      return result;

    } catch (error) {
      await this.repo.update(record.id, {
        status:       RequestStatus.FAILED,
        errorMessage: (error as Error).message,
      });
      throw error;
    }
  }
}
```

---


---

## 5. Resilience — Circuit Breaker, Retry, Rate Limits, Fallback

```typescript
// ✅ Good — complete resilience stack for all third-party calls
import CircuitBreaker from 'opossum';

@Injectable()
export class ResilientHttpClient {

  private breakers = new Map<string, CircuitBreaker>();

  private getBreaker(provider: string): CircuitBreaker {
    if (!this.breakers.has(provider)) {
      const breaker = new CircuitBreaker(
        (fn: () => Promise<unknown>) => fn(), {
          timeout:                  5000,  // treat as failure after 5s
          errorThresholdPercentage: 50,    // open if 50% fail
          resetTimeout:             30000, // try half-open after 30s
          volumeThreshold:          5,     // need 5 requests before evaluating
        }
      );
      // ✅ Log state transitions — know when a provider is struggling
      breaker.on('open',     () => logger.error('Circuit OPEN',     { provider }));
      breaker.on('halfOpen', () => logger.warn ('Circuit HALF-OPEN',{ provider }));
      breaker.on('close',    () => logger.info ('Circuit CLOSED',   { provider }));
      this.breakers.set(provider, breaker);
    }
    return this.breakers.get(provider)!;
  }

  async callWithResilience<T>(
    provider:  string,
    operation: string,
    fn:        () => Promise<T>,
    options?: {
      maxRetries?: number;
      fallback?:   () => T;
    },
  ): Promise<T> {
    const maxRetries = options?.maxRetries ?? 3;
    const breaker    = this.getBreaker(provider);

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await breaker.fire(fn) as T;

      } catch (error) {
        const isLastAttempt  = attempt === maxRetries;
        const isTransient    = this.isTransientError(error);

        // ✅ Handle rate limit — respect Retry-After header
        if (this.isRateLimitError(error)) {
          const retryAfterSecs = this.getRetryAfter(error) ?? 60;
          logger.warn('Rate limited — waiting', { provider, operation, retryAfterSecs });
          if (!isLastAttempt) {
            await sleep(retryAfterSecs * 1000);
            continue;
          }
        }

        if (!isTransient || isLastAttempt) {
          // ✅ Return fallback if configured — degrade gracefully instead of crashing
          if (options?.fallback) {
            logger.warn('Using fallback', { provider, operation, attempt });
            return options.fallback();
          }
          throw error;
        }

        // ✅ Exponential backoff with jitter — prevents thundering herd
        const delayMs = Math.min(
          1000 * Math.pow(2, attempt - 1) + Math.random() * 300,
          15_000 // cap at 15 seconds
        );
        logger.warn('Retrying after transient error', { provider, operation, attempt, delayMs });
        await sleep(delayMs);
      }
    }
    throw new Error('Unreachable');
  }

  private isTransientError(err: unknown): boolean {
    if (!(err instanceof Error)) return false;
    const status = (err as any).status ?? (err as any).statusCode ?? 0;
    return status === 0 || status === 429 || status >= 500; // network, rate limit, server errors
  }

  private isRateLimitError(err: unknown): boolean {
    return ((err as any).status ?? (err as any).statusCode) === 429;
  }

  private getRetryAfter(err: unknown): number | null {
    const header = (err as any).headers?.['retry-after'];
    return header ? Number(header) : null;
  }
}
```

---


---

## 6. Async & Parallel Processing for High-Volume Operations

```typescript
// ❌ Bad — sequential loop for bulk operations: slow + hits rate limits
async function sendBulkEmails(userIds: string[]): Promise<void> {
  for (const userId of userIds) {
    const user = await userRepo.findById(userId);
    await emailGateway.send(user.email, 'newsletter', {});
    // 1000 users × 200ms each = 200 seconds! Also likely hits rate limit
  }
}

// ✅ Good — batched + async queue for bulk operations
async function sendBulkEmails(userIds: string[]): Promise<void> {
  // Enqueue all — workers process at safe rate respecting API limits
  for (const userId of userIds) {
    await queue.enqueue('email.send', { userId, template: 'newsletter' });
  }
  logger.info('Bulk email jobs enqueued', { count: userIds.length });
}

// Worker processes at controlled rate
// workers/email.worker.ts
worker.process('email.send', { concurrency: 10 }, async (job) => {
  await emailWrapper.send(job.data);
});

// ✅ Good — batched parallel calls for data fetching (respect rate limits)
async function processInBatches<T, R>(
  items:      T[],
  batchSize:  number,
  processFn:  (batch: T[]) => Promise<R[]>,
  delayMs:    number = 500, // pause between batches to avoid rate limits
): Promise<R[]> {
  const results: R[] = [];
  for (let i = 0; i < items.length; i += batchSize) {
    const batch       = items.slice(i, i + batchSize);
    const batchResult = await processFn(batch);
    results.push(...batchResult);

    const isLastBatch = i + batchSize >= items.length;
    if (!isLastBatch) await sleep(delayMs);
  }
  return results;
}

// ✅ Good — parallel execution for independent calls
async function enrichOrders(orders: Order[]): Promise<EnrichedOrder[]> {
  // Process 5 at a time — respects rate limits, faster than sequential
  return processInBatches(orders, 5, async (batch) =>
    Promise.all(batch.map(order => this.externalService.enrich(order)))
  );
}

// ✅ Good — async HTTP endpoint: accept and respond immediately
@Post('reports/generate')
@UseGuards(JwtAuthGuard)
async generateReport(@Body() dto: ReportDto, @CurrentUser() user: AuthUser) {
  const jobId = await this.queue.enqueue('report.generate', {
    userId: user.id, params: dto,
  });
  // Return 202 immediately — caller polls /reports/jobs/:jobId for status
  return { jobId, status: 'queued', pollUrl: `/reports/jobs/${jobId}` };
}
```

---


---

## 7. Input/Output Validation at Integration Boundaries

Validate both what you send AND what you receive. Third-party schemas change without notice.

```typescript
// ❌ Bad — response used directly without validation
async function getExchangeRate(from: string, to: string): Promise<number> {
  const resp = await currencyApi.get(`/latest?base=${from}&symbols=${to}`);
  return resp.data.rates[to]; // undefined if API changed schema — silent bug!
}

// ✅ Good — schema validation on both sides of the boundary
import { z } from 'zod';

// Define expected response shape
const ExchangeRateResponseSchema = z.object({
  base:  z.string(),
  date:  z.string(),
  rates: z.record(z.string(), z.number().positive()),
});

async function getExchangeRate(from: string, to: string): Promise<number> {
  // Validate outbound parameters
  if (!['USD', 'EUR', 'GBP', 'INR'].includes(from)) {
    throw new ValidationError('from', `Unsupported currency: ${from}`);
  }

  const raw    = await currencyApiWrapper.get(`/latest?base=${from}&symbols=${to}`);
  const parsed = ExchangeRateResponseSchema.safeParse(raw.data);

  if (!parsed.success) {
    logger.error('Currency API schema mismatch — provider may have changed API', {
      provider: 'fixer.io',
      issues:   parsed.error.flatten(),
    });
    throw new IntegrationSchemaError('Unexpected response from currency API');
  }

  const rate = parsed.data.rates[to];
  if (rate === undefined) {
    throw new IntegrationDataError(`Rate for ${to} not in response`);
  }
  return rate;
}

// ✅ Good — validate before sending (wrapper enforces contract both ways)
const SendEmailInputSchema = z.object({
  to:         z.string().email(),
  subject:    z.string().min(1).max(200),
  templateId: z.string().regex(/^d-[a-f0-9]+$/), // SendGrid template ID format
  data:       z.record(z.string(), z.unknown()),
});

class SendGridWrapper implements IEmailGateway {
  async send(input: SendEmailInput): Promise<void> {
    const validated = SendEmailInputSchema.parse(input); // throws if invalid
    await sgMail.send({ ...validated, from: this.config.defaultFrom });
  }
}
```

---


---

