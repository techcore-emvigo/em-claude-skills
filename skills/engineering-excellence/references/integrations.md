# Third-Party Integrations Best Practices

## Universal Integration Principles

### Credentials & Secrets
- Store all API keys, tokens, and client secrets in a secrets manager (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager)
- Never hardcode or commit credentials — rotate immediately if leaked
- Use separate credentials per environment (dev/staging/prod)
- Set minimum required scopes/permissions for OAuth apps and API keys
- Rotate credentials on a schedule; audit unused credentials quarterly

### Resilience
- All external calls MUST have a timeout — default: 5s for sync, 30s for async
- Implement retry with exponential backoff + jitter for transient failures (5xx, network errors)
- Do NOT retry on 4xx errors (client errors are not transient)
- Use a circuit breaker for dependencies in the critical path (Resilience4j, go-resilience, etc.)
- Implement bulkhead isolation: don't let one slow integration starve the rest of the app
- Store enough state to resume/replay if an integration call fails

```python
# ✅ Good: retry with backoff
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=1, max=10))
def call_payment_api(payload):
    response = httpx.post(PAYMENT_URL, json=payload, timeout=10.0)
    response.raise_for_status()
    return response.json()
```

### Error Handling
- Map external error codes to internal domain errors — don't leak third-party error shapes to clients
- Log the full request ID / transaction ID returned by the third-party for support tracing
- Handle partial success responses explicitly
- Alert on sustained error rate increases from any integration

---

## Webhooks

### Receiving Webhooks
- **Always verify webhook signatures** — use the provider's HMAC/signature scheme
- Return `200 OK` immediately; process asynchronously via a queue
- Be idempotent: webhooks can be delivered more than once — use the event ID for deduplication
- Validate the event type and payload schema before processing
- Log all incoming webhook payloads for replay/debugging

```python
# ✅ Good: Stripe webhook signature verification
import stripe
from fastapi import Request, HTTPException

async def handle_stripe_webhook(request: Request):
    payload = await request.body()
    sig_header = request.headers.get("stripe-signature")
    try:
        event = stripe.Webhook.construct_event(payload, sig_header, WEBHOOK_SECRET)
    except stripe.error.SignatureVerificationError:
        raise HTTPException(status_code=400, detail="Invalid signature")
    # enqueue event for processing
    await queue.enqueue("stripe_event", event)
    return {"status": "ok"}
```

### Sending Webhooks
- Sign outbound payloads with HMAC-SHA256 — include the signature in a header
- Include a unique event ID and timestamp in every payload
- Implement delivery retry with exponential backoff
- Track delivery status; allow consumers to re-fetch events via an event log endpoint
- Document your signature scheme and provide verification examples

---

## OAuth 2.0 / SSO

- Use PKCE for all public/mobile clients — never authorization code flow without PKCE
- Store `access_token` in memory (not localStorage); store `refresh_token` in httpOnly cookie
- Validate `state` parameter to prevent CSRF
- Validate `id_token` signature, `aud`, `iss`, and `exp` claims on every SSO login
- Implement token refresh silently in the background; handle 401 responses by refreshing
- Revoke tokens on logout via the provider's revocation endpoint
- Short-lived access tokens (15–60 min); longer refresh tokens (days/weeks) with rotation

---

## Payment Gateways (Stripe, PayPal, etc.)

- Never log full card numbers, CVVs, or raw PAN data — log only last 4 digits and card type
- Use provider-hosted fields / SDKs (Stripe Elements, Braintree Drop-in) — raw card data should never touch your server
- Idempotency keys on every charge/refund call
- Store `payment_intent_id` / `charge_id` for reconciliation, not card data
- Test all failure scenarios: declined, insufficient funds, 3DS required, network failure
- PCI compliance scope: if using hosted fields, you're SAQ-A; handling raw cards = SAQ-D

---

## Email / SMS Providers (SendGrid, Twilio, etc.)

- Rate limit outbound sends per user to prevent abuse
- Unsubscribe links required in all marketing emails (CAN-SPAM, GDPR)
- Handle bounce and spam complaint webhooks — suppress addresses that hard-bounce
- Validate email addresses before sending (format + MX record check for critical flows)
- SMS: validate phone number format (E.164); handle opt-out (`STOP`) replies
- Log send attempts and delivery status; alert on high bounce/failure rates

---

## Cloud SDKs (AWS, GCP, Azure)

- Use managed identity / service accounts instead of access keys wherever possible
- If access keys are required: store in secrets manager, rotate every 90 days, never in environment variables in code
- Use least-privilege IAM policies — generate from `CloudTrail` access analysis
- Enable SDK retries with backoff — most SDKs have this built in, just configure it
- Use SDK-level timeouts and max retries; don't use defaults blindly
- Log cloud API calls with correlation IDs for tracing

---

## Integration Testing Checklist
- [ ] Timeout behavior tested (what happens when the third-party is slow/unreachable)
- [ ] Retry behavior tested (idempotent?  correct backoff?)
- [ ] Webhook signature verification tested with invalid signatures
- [ ] OAuth token refresh flow tested
- [ ] All error codes from the third-party handled explicitly
- [ ] No credentials in test fixtures or CI logs
- [ ] Contract tests or recorded API responses used (not live calls in unit tests)

---

## Release-Time Integration Checklist

| Gap | What to Look For | Severity |
|---|---|---|
| New third-party integration not documented | Knowledge siloed; maintainability lost | 🟠 |
| Integration method (OAuth2, API key, webhook, SDK) not reviewed | Insecure or brittle integration method used | 🟠 |
| Third-party integration not tested under load | Integrations fail at production scale | 🟠 |
| Third-party API calls not logged (request/response metadata) | Integration issues invisible in production | 🟠 |
| Error handling and timeouts not implemented (no retry, no circuit breaker) | One failing integration takes down the feature | 🔴 |
| Data privacy policy of third-party not reviewed | GDPR/CCPA compliance violation | 🔴 |
| Rate limits of third-party not reviewed vs expected production traffic | Rate limit exceeded under normal load | 🟠 |
| Third-party integrations not monitored for real-time issues | Failures invisible until users complain | 🟠 |
| OAuth2 or equivalent secure auth not used | Credentials exposed as plain API keys | 🟠 |

---

## Payment Gateway Integration — Critical Rules

### Stripe / GooglePay Must Be Backend-Only

```typescript
// ❌ Critical — payment SDK called directly from a React Native screen
// screens/CheckoutScreen.tsx
import Stripe from 'stripe';
const stripe = new Stripe(STRIPE_SECRET_KEY);  // secret key in mobile bundle!

async function handlePayment() {
  await stripe.paymentIntents.create({ amount, currency }); // on device!
}

// ❌ Critical — GooglePay integrated directly in screen component
import { GooglePayButton } from '@google-pay/button-react-native';
<GooglePayButton
  apiKey={GOOGLE_PAY_API_KEY}   // key exposed in bundle
  onPress={directlyChargeCard}  // no backend — insecure
/>

// ✅ Good — payment always flows through your backend service
// screens/CheckoutScreen.tsx  — only creates a payment intent via YOUR API
async function initiatePayment(cartId: string) {
  const { clientSecret } = await apiClient.post('/payments/intent', { cartId });
  // clientSecret is safe to pass to Stripe SDK on device (not a secret key)
  await stripe.confirmPayment({ paymentMethodType: 'Card', clientSecret });
}

// backend: payment.controller.ts  — only here does Stripe API get called with secret key
@Post('payments/intent')
@UseGuards(JwtAuthGuard)
async createPaymentIntent(@Body() dto: CreatePaymentIntentDto, @CurrentUser() user: AuthUser) {
  // Stripe secret key NEVER leaves the backend
  const intent = await this.stripeService.createIntent(dto.amount, dto.currency, user.id);
  return { clientSecret: intent.client_secret }; // only client_secret returned
}
```

### Never Log Card or Bank Data

```typescript
// ❌ Critical — card or bank data in logs
logger.info('Payment processed', {
  cardNumber: '4111111111111111',  // PCI violation
  cvv:        '123',              // PCI violation
  bankAccount: 'ACC123456',       // sensitive
  sortCode:    '12-34-56',        // sensitive
});

// ❌ Critical — full Stripe response logged (may contain sensitive metadata)
logger.debug('Stripe response', { stripeResponse });  // may have billing address etc.

// ✅ Good — log only safe references
logger.info('Payment intent created', {
  paymentIntentId: intent.id,          // Stripe's own reference ID
  amount:          intent.amount,      // amount is fine
  currency:        intent.currency,
  userId:          user.id,
  lastFour:        card.last4,         // last 4 digits only
  cardBrand:       card.brand,         // 'visa', 'mastercard' — fine
  traceId:         req.traceId,
});

// ✅ Good — never store card data on your server
// Use Stripe's tokenisation: collect card via Stripe Elements (frontend)
// Your server only receives a payment method token — never raw card numbers
```

### Payment Handler Service Pattern

```typescript
// ✅ Good — dedicated payment service class (not inline in screen/controller)
// services/payment.service.ts
@Injectable()
export class PaymentService {
  constructor(
    @Inject(STRIPE_CLIENT) private stripe: Stripe,
    private orderRepo: IOrderRepository,
    private auditService: AuditService,
  ) {}

  async createPaymentIntent(
    orderId: string,
    amount:  number,
    currency: string,
    userId:  string,
  ): Promise<{ clientSecret: string }> {
    logger.info('Creating payment intent', { orderId, amount, currency, userId });

    const intent = await this.stripe.paymentIntents.create({
      amount:   Math.round(amount * 100), // Stripe uses smallest currency unit
      currency,
      metadata: { orderId, userId },       // safe metadata — no PII
      idempotency_key: `order_${orderId}`, // prevent duplicate charges
    });

    await this.auditService.log({
      actorId:      userId,
      action:       'payment.intent_created',
      resourceType: 'order',
      resourceId:   orderId,
      metadata:     { paymentIntentId: intent.id, amount, currency },
    });

    logger.info('Payment intent created', { paymentIntentId: intent.id, orderId });
    return { clientSecret: intent.client_secret };
  }
}
```

---

## Gap Detection Table (Additions)

| Gap | What to Look For | Severity |
|---|---|---|
| Stripe secret key in mobile bundle | `new Stripe(process.env.STRIPE_SECRET_KEY)` in React Native | 🔴 |
| Payment SDK called directly from screen | Stripe/GooglePay API calls in component, not service class | 🔴 |
| Card number or CVV in log | `cardNumber`, `cvv`, `accountNumber` in any log statement | 🔴 |
| Full Stripe/PayPal response logged | `logger.debug(stripeFullResponse)` — may contain sensitive data | 🔴 |
| No idempotency key on charge | `paymentIntents.create()` without `idempotency_key` | 🔴 |
| No payment audit trail | Payment created/refunded with no audit log entry | 🟠 |
| Payment handler not in service class | Payment logic inside controller or component | 🟠 |
