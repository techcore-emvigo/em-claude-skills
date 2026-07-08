# Third-Party Patterns — Health, Idempotency & Testing

Covers: health monitoring, duplicate request prevention (idempotency),
testing strategy (mocks, contract tests, negative scenarios).

---

## 8. Health Monitoring

```typescript
// ✅ Good — active health check per provider
@Injectable()
export class ThirdPartyHealthMonitor {

  private status = new Map<string, { healthy: boolean; lastCheck: Date; reason?: string }>();

  @Cron('*/30 * * * * *') // every 30 seconds
  async checkAllProviders(): Promise<void> {
    await Promise.allSettled([
      this.check('stripe',   () => fetch('https://status.stripe.com/api/v2/status.json')
                                    .then(r => r.json())
                                    .then(d => { if (d.status.indicator !== 'none') throw new Error(d.status.description); })),
      this.check('sendgrid', () => fetch('https://status.sendgrid.com/api/v2/status.json')
                                    .then(r => r.json())
                                    .then(d => { if (d.status.indicator !== 'none') throw new Error(d.status.description); })),
      this.check('twilio',   () => fetch('https://status.twilio.com/api/v2/status.json')
                                    .then(r => r.json())
                                    .then(d => { if (d.status.indicator !== 'none') throw new Error(d.status.description); })),
    ]);
  }

  private async check(provider: string, fn: () => Promise<void>): Promise<void> {
    try {
      await fn();
      const wasUnhealthy = !this.status.get(provider)?.healthy;
      this.status.set(provider, { healthy: true, lastCheck: new Date() });
      if (wasUnhealthy) {
        logger.info('Third-party provider recovered', { provider });
        // Send recovery alert
      }
    } catch (error) {
      const wasHealthy = this.status.get(provider)?.healthy !== false;
      this.status.set(provider, { healthy: false, lastCheck: new Date(), reason: (error as Error).message });
      if (wasHealthy) {
        logger.error('Third-party provider degraded', { provider, reason: (error as Error).message });
        // Send alert to on-call
      }
    }
  }

  getStatus(provider: string): boolean {
    return this.status.get(provider)?.healthy ?? true; // default to healthy if never checked
  }
}

// ✅ Expose in /ready endpoint for load balancer health checks
@Get('/ready')
readinessCheck() {
  return {
    database: this.dbHealth.isHealthy(),
    stripe:   this.thirdPartyHealth.getStatus('stripe'),
    sendgrid: this.thirdPartyHealth.getStatus('sendgrid'),
  };
}
```

---


---

## 9. Idempotency — Duplicate Call Prevention

```typescript
// ✅ Good — idempotency key on every mutating third-party call
class StripeWrapper {
  async charge(orderId: string, amount: number): Promise<string> {
    const intent = await this.stripe.paymentIntents.create(
      { amount, currency: 'usd' },
      { idempotencyKey: `order_charge_${orderId}` } // Stripe deduplicates on this
    );
    return intent.id;
  }
}

// ✅ Good — DB-level deduplication for email/SMS (idempotent sends)
async function sendWelcomeEmailOnce(userId: string, email: string): Promise<void> {
  const alreadySent = await emailLogRepo.findOne({
    where: { userId, template: 'welcome', status: 'sent' }
  });
  if (alreadySent) {
    logger.info('Duplicate welcome email blocked', { userId });
    return;
  }
  await emailWrapper.send({ to: email, templateId: 'd-welcome123', data: {} });
  await emailLogRepo.save({ userId, template: 'welcome', status: 'sent', sentAt: new Date() });
}

// ✅ Good — distributed lock for concurrent webhook deduplication
async function processWebhookOnce(eventId: string, fn: () => Promise<void>): Promise<void> {
  const lockKey  = `webhook:${eventId}`;
  const acquired = await redis.set(lockKey, '1', 'NX', 'EX', 300);
  if (!acquired) {
    logger.info('Duplicate webhook ignored', { eventId });
    return;
  }
  try {
    await fn();
  } finally {
    await redis.del(lockKey);
  }
}
```

---


---

## Gap Detection Table — Implementation & Operations

| Gap | What to Look For | Severity |
|---|---|---|
| No wrapper interface | Third-party SDK imported directly in service/controller/screen | 🔴 |
| Direct SDK call in mobile UI | Stripe/GooglePay called from component, not service | 🔴 |
| No circuit breaker | External call with no failure threshold or recovery | 🔴 |
| No timeout on HTTP call | External call with no timeout option set | 🟠 |
| Retry on 4xx errors | Client errors retried — wastes quota | 🟠 |
| No exponential backoff + jitter | Fixed-interval retry causes thundering herd | 🟠 |
| No fallback | App fails completely when one provider goes down | 🟠 |
| No idempotency key on mutating call | Duplicate charges/sends on retry | 🔴 |
| Third-party response not schema-validated | Silent failure when API shape changes | 🟠 |
| Rate limit (429) not handled | Provider account suspended | 🟠 |
| No audit log for third-party calls | Failures impossible to diagnose | 🔴 |
| Audit log on critical path | DB write inside main request adds latency | 🟠 |
| Request status not tracked in DB | No replay or debugging capability | 🟠 |
| No health monitoring of provider | Outage discovered by user complaints | 🟠 |
| No mock in unit tests | Tests call real external API | 🟠 |
| Duplicate call scenario not tested | Idempotency untested — double-charge possible | 🔴 |
| Production API key in any test | Live charges in CI — billing risk | 🔴 |
