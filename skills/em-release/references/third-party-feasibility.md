# Third-Party Integration — Feasibility & Governance

Covers: pre-development feasibility assessment, POC requirements, cost/limitation
communication, team responsibilities, documentation structure.

---

## Overview

Third-party integrations must go through a structured lifecycle:
**Feasibility → Design → Implementation → Testing → Release → Operations**

Every stage has defined responsibilities for Architect, Tech Lead, BA, Developer, and QA.
Skipping any stage creates hidden costs, security risks, and unmaintainable code.

---


---

## 1. Feasibility Assessment — Before Any Code

Run this checklist before committing to any third-party. Undiscovered limitations found
mid-sprint cost 10x more to fix than those found during feasibility.

**Owner: Architect + Tech Lead + BA — must be completed before sprint planning.**

```markdown

---

## 10. Testing Strategy

```typescript
// ✅ Good — unit tests mock the interface (never call real APIs)
describe('OrderService.checkout', () => {

  let service:      OrderService;
  let mockPayment:  jest.Mocked<IPaymentGateway>;

  beforeEach(() => {
    mockPayment = {
      createPaymentIntent: jest.fn(),
      capturePayment:      jest.fn(),
      refund:              jest.fn(),
    };
    service = new OrderService(mockPayment, mockOrderRepo);
  });

  // ✅ Happy path
  it('returns clientSecret on success', async () => {
    mockPayment.createPaymentIntent.mockResolvedValue({
      clientSecret: 'pi_test_secret_123',
      intentId:     'pi_test_123',
    });
    const result = await service.checkout('cart_1', 'user_1');
    expect(result.clientSecret).toBe('pi_test_secret_123');
  });

  // ✅ All negative scenarios — each error type tested separately
  it('handles card declined', async () => {
    mockPayment.createPaymentIntent.mockRejectedValue(
      new PaymentDeclinedError('Card declined', { code: 'card_declined' })
    );
    await expect(service.checkout('cart_1', 'user_1')).rejects.toThrow(PaymentDeclinedError);
  });

  it('handles rate limit', async () => {
    mockPayment.createPaymentIntent.mockRejectedValue(new PaymentRateLimitError('Rate limited'));
    await expect(service.checkout('cart_1', 'user_1')).rejects.toThrow(PaymentRateLimitError);
  });

  it('handles timeout', async () => {
    mockPayment.createPaymentIntent.mockRejectedValue(new TimeoutError('5s timeout'));
    await expect(service.checkout('cart_1', 'user_1')).rejects.toThrow();
  });

  it('handles service unavailable', async () => {
    mockPayment.createPaymentIntent.mockRejectedValue(new Error('Service unavailable'));
    await expect(service.checkout('cart_1', 'user_1')).rejects.toThrow();
  });

  // ✅ Duplicate call scenario
  it('does not double-charge on retry', async () => {
    mockPayment.createPaymentIntent.mockResolvedValue({ clientSecret: 'pi_x', intentId: 'pi_1' });
    await service.checkout('cart_1', 'user_1');
    await service.checkout('cart_1', 'user_1'); // same cart again
    // Idempotency key ensures Stripe only charges once
    expect(mockPayment.createPaymentIntent).toHaveBeenCalledWith(
      expect.anything(), expect.anything(),
      expect.objectContaining({ orderId: expect.any(String) })
    );
  });
});

// ✅ Good — integration test uses test-mode credentials (never production)
describe('StripeWrapper integration', () => {
  const wrapper = new StripePaymentGateway({
    stripeSecretKey: process.env.STRIPE_TEST_KEY!, // sk_test_... only
  } as AppConfig);

  it('creates payment intent in test mode', async () => {
    const result = await wrapper.createPaymentIntent(1000, 'usd', { orderId: 'test_001' });
    expect(result.intentId).toMatch(/^pi_/);
    expect(result.clientSecret).toContain('_secret_');
  });
});
```

---


---

## 11. Team Responsibilities

```markdown

---

## Gap Detection Table — Feasibility & Documentation

| Gap | What to Look For | Severity |
|---|---|---|
| No POC before implementation | Integration committed based on docs only | 🟠 |
| Requirements coverage undocumented | No record of what % the third-party satisfies | 🟠 |
| Cost not shared with client upfront | Client discovers billing costs post-launch | 🔴 |
| Limitations not communicated to client | Scalability/feature gaps found in UAT | 🟠 |
| Tech debt not in backlog | Integration gaps not tracked as stories | 🟠 |
| Beta/deprecated API in use | Endpoint subject to breaking changes | 🟠 |
| Architect approval missing | Package/wrapper used without architecture review | 🟠 |
| BA flows exceed third-party capabilities | UI designed around impossible API behaviours | 🟠 |
| No integration guide in docs | Auth, error codes, limits known only to original dev | 🟠 |
| No client sign-off on limitations | Assumptions not formally agreed | 🟠 |
| Data retention policy undocumented | GDPR compliance unclear for third-party data | 🔴 |
