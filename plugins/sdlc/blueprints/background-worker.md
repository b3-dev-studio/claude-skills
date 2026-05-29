---
artifact: blueprint
name: background-worker
version: 1
tags: [async, queue, job, worker, background, task]
---

# Blueprint: Background Worker

## When to Use

Apply this blueprint when building a feature that:
- Must run outside the HTTP request lifecycle
- Is triggered by a queue, schedule, or event
- Involves work that is too slow, too unreliable, or too resource-intensive for a synchronous response
- Requires retry logic, dead-letter handling, or rate limiting

Do not apply when: the work completes in milliseconds and the caller must wait for the result, the feature is a pure HTTP resource, or the trigger is a domain event on an internal bus (use `event-driven-handler` instead).

---

## Canonical Component Graph

```
C1 — Producer
  │  Owns: creating and enqueuing job payloads
  │  Does not own: job processing, retry logic, business logic
  │  Lives in: the service or controller that triggers the work
  │
  ▼  (queue boundary)
C2 — Worker
  │  Owns: dequeuing, concurrency control, retry/backoff, dead-letter routing
  │  Does not own: the business logic of the job itself
  │
  ▼
C3 — Job Handler
     Owns: the actual business logic for one job type
     Does not own: queue mechanics, retry decisions
     One handler class per job type
```

The queue boundary between C1 and C2 is the key structural property. C1 and C2 may live in different processes. C3 is instantiated by C2 and must be stateless.

---

## Required Interfaces (abstract)

**C1 — Producer**
- An enqueue method that accepts a typed job payload and returns a job ID or acknowledgement
- Must not block on job completion
- Must validate the payload before enqueuing

**C2 — Worker**
- Registration mechanism that binds a queue name to a handler class
- Concurrency configuration
- Retry policy configuration (max attempts, backoff strategy)
- Dead-letter queue or failure hook

**C3 — Job Handler**
- A single `execute` (or equivalent) method accepting the typed job payload
- Returns void or a result indicating success
- Throws typed errors to signal retriable vs. non-retriable failures
- Stateless — safe to instantiate per job

---

## Adaptation Questions

The bootstrap generator must answer these by surveying the codebase.

1. What queue or job library is in use? (e.g., BullMQ, Sidekiq, Celery, Temporal, pg-boss)
2. How are workers registered and started? (separate process, worker file, framework convention)
3. What is the job payload format? (plain object, typed class, serialized JSON)
4. How is retry policy configured? (library config, decorator, per-handler)
5. How are permanently failed jobs handled? (dead-letter queue, database record, alert)
6. How is the queue connection configured and where are credentials stored?
7. Is there an existing job handler to use as a representative example? Trace it end-to-end.
8. How is job progress or status tracked if at all?
9. Are jobs idempotent by convention? Is there a deduplication mechanism?
10. Where does the producer live relative to the service layer? (inside a service, separate class, inline)

---

## Known Anti-Patterns

These must appear in the bootstrap's Constraints section, confirmed against the actual codebase.

- **Synchronous producer:** enqueue call blocks until job completes, defeating the purpose of the queue.
- **Business logic in the worker:** C2 contains domain logic instead of delegating to C3. The worker should only handle queue mechanics.
- **Unbounded retries:** no maximum retry count, causing permanently broken jobs to retry forever.
- **Non-idempotent handlers:** handler has side effects that produce incorrect state if run more than once. Handlers must be safe to re-run.
- **Swallowed errors:** exceptions caught and logged but not re-thrown, causing the queue to mark a failed job as succeeded.
- **Payload mutation:** producer modifies shared state after enqueuing, causing job to run against stale or different data than intended.
- **God handler:** one handler class handles multiple unrelated job types. One handler per job type.
