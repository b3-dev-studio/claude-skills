---
artifact: blueprint
name: webhook-receiver
version: 1
tags: [webhook, inbound, callback, http, integration, external]
---

# Blueprint: Webhook Receiver

## When to Use

Apply this blueprint when building a feature that:
- Accepts inbound HTTP POST callbacks from an external service (Stripe, GitHub, Twilio, Slack, etc.)
- Must verify the authenticity of inbound payloads via a signature or token
- Must respond to the external service within a tight deadline (typically 3–10 seconds)
- Triggers processing that may take longer than the deadline allows

Do not apply when: the HTTP endpoint is a first-party API (use `rest-api-resource`), the trigger is internal (use `event-driven-handler`), or there is no signature verification requirement.

---

## Canonical Component Graph

```
C1 — Webhook Controller
  │  Owns: HTTP route, raw body capture, signature verification,
  │        immediate 2xx response
  │  CRITICAL: must respond before the provider's timeout (typically 5s)
  │  Does not own: payload parsing, business logic, async processing
  │
  ▼
C2 — Webhook Dispatcher
  │  Owns: parsing and validating the event payload, routing by event type,
  │        enqueuing for async processing, idempotency checks
  │  Does not own: HTTP mechanics, business logic
  │
  ▼
C3 — Event-Specific Processor
     Owns: the business logic for one webhook event type
     One processor class per event type
     Runs asynchronously — not in the HTTP request lifecycle
```

The synchronous/asynchronous boundary between C2 and C3 is the key structural property. C1 and C2 run in the request. C3 runs in a background worker. If C3 is fast and idempotent, it may run inline for simplicity, but the architecture must make the boundary explicit.

---

## Required Interfaces (abstract)

**C1 — Webhook Controller**
- Raw body must be captured before any JSON parsing (required for HMAC verification)
- Signature verification must happen before any payload processing
- Returns 200 immediately on valid signature, regardless of processing outcome
- Returns 401 on signature failure — nothing else
- Never returns 5xx on processing errors (causes provider retry storms)

**C2 — Webhook Dispatcher**
- Accepts verified raw payload + event type header
- Parses into typed event object per event type
- Checks idempotency key (event ID) before processing — skip if already seen
- Routes to correct C3 processor or enqueues for async execution
- Records receipt of event (for debugging and replay)

**C3 — Event-Specific Processor**
- A single `process` (or equivalent) method accepting the typed event payload
- Idempotent — safe to run more than once for the same event ID
- Throws on failure — does not swallow errors

---

## Adaptation Questions

The bootstrap generator must answer these by surveying the codebase.

1. Is there an existing webhook receiver to use as a representative example? Trace it end-to-end.
2. How is HMAC/signature verification implemented? Is there a shared utility?
3. Where are webhook secrets stored and how are they accessed?
4. How is the raw request body captured before framework JSON parsing middleware?
5. How are webhook events queued for async processing? (same queue as background-worker blueprint?)
6. Is there an idempotency store? (Redis, database table with event IDs)
7. Where are webhook event receipts logged?
8. How are event type headers or payload fields used to route to the correct processor?
9. What is the response strategy for unknown event types? (200 and ignore, or 422)
10. Is there a webhook event replay mechanism for debugging?

---

## Known Anti-Patterns

These must appear in the bootstrap's Constraints section, confirmed against the actual codebase.

- **Synchronous heavy processing:** C3 logic runs inline in the HTTP request, risking provider timeouts and retry storms.
- **Signature verification after parsing:** JSON parsing before HMAC verification allows payload tampering. Raw body must be captured first.
- **5xx on processing failure:** returning a server error causes the provider to retry indefinitely. Always return 200 after verifying the signature; handle processing failures out-of-band.
- **No idempotency:** provider will retry on network errors. Processing the same event twice must be safe.
- **Shared processor for all event types:** one processor branching on event type. One processor per event type.
- **Hardcoded secrets:** webhook secrets referenced directly in code rather than from environment/config.
- **No event receipt record:** no log or database record of inbound events makes debugging and replay impossible.
