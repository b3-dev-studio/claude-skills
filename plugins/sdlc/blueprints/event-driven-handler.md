---
artifact: blueprint
name: event-driven-handler
version: 1
tags: [event, pub-sub, listener, handler, domain-event, message]
---

# Blueprint: Event-Driven Handler

## When to Use

Apply this blueprint when building a feature that:
- Reacts to something that happened elsewhere in the system
- Should be decoupled from the code that caused the event
- Involves cross-cutting concerns triggered by domain state changes (notifications, audit logs, cache invalidation, side effects)

Do not apply when: the feature initiates work on a schedule or queue (use `background-worker`), the trigger is an inbound HTTP callback from an external service (use `webhook-receiver`), or the caller must wait for the result synchronously.

---

## Canonical Component Graph

```
C1 — Event Subscriber
  │  Owns: registering with the event bus, receiving raw event messages,
  │        deserialising and validating the event payload
  │  Does not own: business logic, side effects
  │
  ▼
C2 — Event Handler
  │  Owns: the business logic for responding to one event type
  │        orchestrating services and repositories
  │  Does not own: event bus mechanics, serialisation
  │
  ▼
C3 — (optional) Side Effect Services
     Owns: external calls, notifications, secondary writes
     Injected into C2 — not owned by the handler directly
```

One subscriber per event type. One handler per event type. If multiple teams need to react to the same event, they each own their own subscriber + handler pair — they do not share one handler.

---

## Required Interfaces (abstract)

**C1 — Event Subscriber**
- Registration mechanism that binds an event type/topic to a handler
- Deserialises raw event data into a typed event object
- Validates the event schema before dispatching to C2
- Handles bus-level errors (connection loss, timeout) independently of handler errors

**C2 — Event Handler**
- A single `handle` (or equivalent) method accepting the typed event object
- Returns void
- Throws to signal failure — does not catch and swallow
- Stateless between invocations

**C3 — Side Effect Services**
- Standard service interfaces already existing in the codebase
- Injected into C2 via constructor — C2 does not instantiate them

---

## Adaptation Questions

The bootstrap generator must answer these by surveying the codebase.

1. What event bus or pub/sub system is in use? (EventEmitter, Redis pub/sub, Kafka, RabbitMQ, internal domain event dispatcher)
2. How are event types defined and named? (string constants, enums, class names)
3. How are subscribers registered? (decorator, explicit registration call, config file, auto-discovery)
4. How are events serialised and deserialised?
5. What happens when a handler throws? (retry, dead-letter, log-and-continue)
6. Is there a transactional outbox pattern or similar guarantee?
7. Is there an existing event handler to trace as a representative example?
8. How are handlers tested in isolation from the event bus?
9. Is there an event schema registry or type definitions file?
10. How is handler ordering enforced if multiple handlers listen to the same event?

---

## Known Anti-Patterns

These must appear in the bootstrap's Constraints section, confirmed against the actual codebase.

- **Synchronous side effects blocking the bus:** handler makes slow external calls inline, starving other events. Slow side effects should be enqueued as background jobs.
- **Shared handler for multiple event types:** one handler class conditionally branches on event type. One handler per event type.
- **Business logic in the subscriber:** C1 contains domain decisions. The subscriber's only job is receive, validate, dispatch.
- **Silent failure:** handler catches all exceptions and logs without re-throwing. The bus cannot retry what it cannot see fail.
- **Circular events:** handler emits an event that triggers itself, directly or indirectly. All event chains must be acyclic.
- **Coupling to event internals:** handler accesses raw event payload fields directly instead of a typed interface. Changes to event structure silently break handlers.
- **Missing idempotency:** handler is not safe to run twice for the same event (at-least-once delivery is the norm for most buses).
