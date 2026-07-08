# Webhooks — Gap Detection

## Receiving Webhook Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No signature verification | Webhook handler processes payload without HMAC verification | 🔴 |
| Signature check on parsed body | Verifying `JSON.stringify(req.body)` instead of raw body bytes | 🔴 |
| Timing-unsafe comparison | `signature === computed` instead of `crypto.timingSafeEqual(...)` | 🟠 |
| Processing before 200 response | DB writes, API calls, or emails done before `res.status(200).send()` | 🔴 |
| No idempotency check | Event processed without checking if `event.id` was already handled | 🔴 |
| No DLQ for failed processing | Failed event processing silently discarded with no retry or DLQ | 🔴 |
| Unknown event types cause error | `default: throw new Error("unknown type")` — should return 200 and log | 🟠 |
| No payload logging | Received webhooks not logged with eventId — impossible to replay/debug | 🟠 |
| No max body size limit | No request size limit — oversized payload attack vector | 🟡 |

## Sending Webhook Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No signature on outbound payload | Payloads sent without HMAC signature header | 🔴 |
| No timestamp in signature | Signature without timestamp — replay attacks possible | 🟠 |
| No retry on delivery failure | Failed delivery not retried — consumers miss events permanently | 🟠 |
| No delivery status tracking | No record of whether a webhook was delivered successfully | 🟡 |
| No replay mechanism | Customers cannot re-trigger a webhook they missed | 🟡 |

## Generation Checklist
- [ ] Parse raw body before JSON parsing; verify HMAC against raw bytes
- [ ] Use `crypto.timingSafeEqual()` for all signature comparisons
- [ ] Return `200 OK` within 3–5 seconds; enqueue for async processing
- [ ] Deduplicate on `eventId` before processing (Redis `SET NX EX`)
- [ ] DLQ for events that fail after max retries
- [ ] Log every received webhook with eventId, type, and processing outcome
- [ ] Outbound: sign with HMAC-SHA256 + timestamp; retry with exponential backoff
