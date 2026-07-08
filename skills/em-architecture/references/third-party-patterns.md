# Third-Party Patterns — Wrapper & Audit Logging

Covers: mandatory wrapper interface architecture, domain error mapping,
async audit logging off the critical path.

---

# Third-Party Patterns — Wrapper, Audit & Resilience

Covers: wrapper interface architecture, async audit logging off critical path,
request status tracking, circuit breaker, retry/backoff, rate limiting,
async/parallel processing, input/output schema validation.

---

# Third-Party Integration — Patterns & Implementation

Covers: wrapper architecture, audit logging, request status tracking, circuit breaker,
retry/backoff, async processing, schema validation, health monitoring, idempotency, testing.

---

## THIRD-PARTY FEASIBILITY DOCUMENT
Provider: _______________  Version: ___  Date: ___  Owner: ___

### Technical Feasibility
[ ] POC built and tested against real API (not documentation-only review)
[ ] Compatible with current tech stack (Node 20, React 18, iOS 16+, etc.)
[ ] Requirements coverage: ___% satisfied — gaps documented below
[ ] All limitations identified and documented
[ ] Key dependencies listed (other libraries, infra, network requirements)
[ ] Stable, versioned API confirmed (not beta or preview endpoint)
[ ] Auth mechanism verified: OAuth2 / API key / JWT (not plain credentials)
[ ] Official SDK or client library available for our stack
[ ] Flows presented to client are achievable within this third-party's capabilities ← BA+TL sign-off

### Scalability & Performance
[ ] Rate limits documented: req/second, req/day, max payload size
[ ] Rate limits sufficient for production traffic at 1x, 3x, 10x load
[ ] Response time SLA matches our API SLA (e.g., < 800ms)
[ ] Scalability ceiling identified and communicated to client in writing
[ ] Load test results captured during POC

### Cost
[ ] Full cost model: per-call, per-seat, per-GB, infrastructure costs
[ ] Cost shared with client BEFORE development starts — client sign-off obtained
[ ] Cost at 3x and 10x traffic modelled (growth cost known upfront)
[ ] Free tier limits and paid upgrade triggers documented

### Support & Stability
[ ] Support channels identified: docs, community, paid support, SLA tiers
[ ] GitHub stars, ratings, issue tracker health reviewed
[ ] Breaking change history reviewed — stability track record assessed
[ ] Client aware of support limitations and escalation options

### Legal & Compliance
[ ] Data privacy policy reviewed (GDPR, CCPA, HIPAA applicability)
[ ] Data residency requirements verified
[ ] Terms of Service allow our specific use case
[ ] Licence compatible with our product distribution

### Client Communication (Must complete BEFORE development)
[ ] All technical challenges and assumptions published to client
[ ] Tech debt gaps communicated — added to backlog as stories
[ ] Client signed off on feasibility document
[ ] BA has documented standard user stories template for this third-party
```

---


---

## 2. Wrapper Architecture — Every Integration Needs One

**Rule: No third-party SDK or API is ever called directly from business logic or UI.**
Every integration lives behind an interface in a dedicated wrapper class or service.

Benefits:
- Business logic is testable without hitting real APIs
- Provider can be swapped by changing one DI binding
- Breaking changes in third-party SDK are isolated to one file
- Audit logging, retry, and circuit breaker can be added in one place

```typescript
// ❌ Bad — business logic tightly coupled to Stripe SDK
// services/order.service.ts
import Stripe from 'stripe';                          // third-party import in business logic!

export class OrderService {
  private stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

  async checkout(cartId: string, userId: string): Promise<void> {
    const intent = await this.stripe.paymentIntents.create({  // direct SDK call
      amount:   cart.total * 100,
      currency: 'usd',
    });
    // OrderService now depends on Stripe's API shape — impossible to unit test
    // Switching from Stripe to PayPal requires changing OrderService
  }
}
```

```typescript
// ✅ Good — 4-layer wrapper architecture

// ─── LAYER 1: Interface in domain layer (zero third-party imports) ───────────
// domain/interfaces/i-payment-gateway.ts
export interface IPaymentGateway {
  createPaymentIntent(
    amount:   number,
    currency: string,
    metadata: Record<string, string>,
  ): Promise<{ clientSecret: string; intentId: string }>;

  capturePayment(intentId: string): Promise<PaymentResult>;
  refund(intentId: string, amount?: number): Promise<RefundResult>;
}

// ─── LAYER 2: Domain error types (no Stripe types leak out) ─────────────────
// domain/errors/payment.errors.ts
export class PaymentDeclinedError   extends AppError {
  constructor(msg: string, meta?: object) { super('PAYMENT_DECLINED', msg, meta); }
}
export class PaymentRateLimitError  extends AppError {
  constructor(msg: string) { super('PAYMENT_RATE_LIMITED', msg); }
}
export class PaymentProcessingError extends AppError {
  constructor(msg: string, opts?: ErrorOptions) { super('PAYMENT_FAILED', msg, undefined, opts); }
}

// ─── LAYER 3: Wrapper in infrastructure layer (ALL Stripe code here) ─────────
// infrastructure/payment/stripe-payment-gateway.ts
import Stripe from 'stripe';

@Injectable()
export class StripePaymentGateway implements IPaymentGateway {

  private readonly stripe: Stripe;

  constructor(@Inject(APP_CONFIG) private config: AppConfig) {
    this.stripe = new Stripe(config.stripeSecretKey, { apiVersion: '2024-04-10' });
  }

  async createPaymentIntent(
    amount:   number,
    currency: string,
    metadata: Record<string, string>,
  ): Promise<{ clientSecret: string; intentId: string }> {
    try {
      logger.info('Creating payment intent', {
        amount,
        currency,
        orderId: metadata.orderId, // safe — not a secret
      });

      const intent = await this.stripe.paymentIntents.create({
        amount:          Math.round(amount * 100), // Stripe uses minor units
        currency,
        metadata,
        idempotency_key: `pi_${metadata.orderId}`, // prevent duplicate charges
      });

      logger.info('Payment intent created', {
        intentId: intent.id,
        orderId:  metadata.orderId,
      });

      return { clientSecret: intent.client_secret!, intentId: intent.id };

    } catch (err) {
      // ✅ Map Stripe error codes to domain errors — never leak Stripe types upward
      if (err instanceof Stripe.errors.StripeCardError) {
        throw new PaymentDeclinedError(err.message, { code: err.code });
      }
      if (err instanceof Stripe.errors.StripeRateLimitError) {
        throw new PaymentRateLimitError('Payment service rate limit reached');
      }
      logger.error('Payment intent creation failed', {
        orderId: metadata.orderId,
        error:   (err as Error).message,
      });
      throw new PaymentProcessingError('Payment processing failed', { cause: err });
    }
  }

  async capturePayment(intentId: string): Promise<PaymentResult> {
    /* ...Stripe capture logic... */
    return {} as PaymentResult;
  }

  async refund(intentId: string, amount?: number): Promise<RefundResult> {
    /* ...Stripe refund logic... */
    return {} as RefundResult;
  }
}

// ─── LAYER 4: Business logic uses interface only — zero Stripe knowledge ─────
// services/order.service.ts
@Injectable()
export class OrderService {
  constructor(
    @Inject(PAYMENT_GATEWAY) private payment: IPaymentGateway, // interface, not Stripe
    private orderRepo: IOrderRepository,
  ) {}

  async checkout(cartId: string, userId: string): Promise<CheckoutResult> {
    const cart  = await this.orderRepo.findCart(cartId);
    const order = Order.create(cart, userId);
    await this.orderRepo.save(order);

    // No Stripe knowledge — just the interface
    const { clientSecret } = await this.payment.createPaymentIntent(
      order.total,
      order.currency,
      { orderId: order.id, userId },
    );
    return { clientSecret, orderId: order.id };
  }
}

// ─── WIRING: one-line provider swap ──────────────────────────────────────────
// payment.module.ts
@Module({
  providers: [
    { provide: PAYMENT_GATEWAY, useClass: StripePaymentGateway },
    // To switch to PayPal:  useClass: PayPalPaymentGateway
    // In unit tests:        useClass: MockPaymentGateway
    OrderService,
  ],
})
export class PaymentModule {}
```

---


---

## 3. Audit Logging — Async, Off the Critical Path

Every third-party call must be logged. Audit writes must be **async** — they must never
add to the response time of the main request.

```typescript
// ❌ Bad — no audit logging; failures impossible to diagnose
async function sendEmail(to: string, subject: string): Promise<void> {
  await sendgrid.send({ to, subject });
  // If this fails at 2 AM, you have nothing to investigate with
}

// ❌ Bad — synchronous audit write blocks the response
async function sendEmail(to: string, subject: string): Promise<void> {
  const result = await sendgrid.send({ to, subject });
  await db.auditLog.create({ provider: 'sendgrid', result }); // ← adds DB latency to every email!
}

// ✅ Good — fire-and-forget async audit; never blocks main flow
@Injectable()
export class ThirdPartyAuditLogger {

  constructor(
    private queue:  AuditQueue,
    private logger: AppLogger,
  ) {}

  // Wrap any third-party call with full audit lifecycle
  async callWithAudit<T>(
    provider:    string,
    operation:   string,
    safeContext: Record<string, unknown>, // sanitised — no secrets, no PII
    fn:          () => Promise<T>,
  ): Promise<T> {
    const requestId = crypto.randomUUID();
    const startedAt = Date.now();

    this.logger.info('Third-party call started', {
      provider, operation, requestId,
      contextKeys: Object.keys(safeContext), // log field names, not values
    });

    try {
      const result    = await fn();
      const durationMs = Date.now() - startedAt;

      // ✅ Fire-and-forget — does NOT block the response
      void this.queue.enqueue('audit.third_party', {
        provider, operation, requestId,
        status: 'success', durationMs,
        timestamp: new Date().toISOString(),
      }).catch(err =>
        this.logger.error('Audit queue write failed', { err: (err as Error).message })
      );

      this.logger.info('Third-party call succeeded', { provider, operation, requestId, durationMs });
      return result;

    } catch (error) {
      const durationMs = Date.now() - startedAt;

      // ✅ Async audit even on failure — never swallow the original error
      void this.queue.enqueue('audit.third_party', {
        provider, operation, requestId,
        status:    'error',
        errorCode: (error as AppError).code ?? 'UNKNOWN',
        durationMs,
        timestamp: new Date().toISOString(),
      }).catch(() => { /* silently absorb audit queue failure */ });

      this.logger.error('Third-party call failed', {
        provider, operation, requestId, durationMs,
        error: (error as Error).message,
      });

      throw error; // always re-throw — never swallow
    }
  }
}

// ✅ Usage in any wrapper class
async createPaymentIntent(amount: number, currency: string, meta: Record<string, string>) {
  return this.auditLogger.callWithAudit(
    'stripe', 'payment_intents.create',
    { amount, currency, orderId: meta.orderId }, // safe context — no card data
    () => this.stripe.paymentIntents.create({ amount, currency, metadata: meta }),
  );
}
```

---


---

