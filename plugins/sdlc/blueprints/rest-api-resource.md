---
artifact: blueprint
name: rest-api-resource
version: 1
tags: [crud, rest, resource, api, endpoint]
---

# Blueprint: REST API Resource

## When to Use

Apply this blueprint when building a feature that:
- Exposes one or more HTTP endpoints for a domain resource
- Requires create, read, update, or delete operations (any subset)
- Has a clear separation between HTTP concerns and business logic
- Persists data to a database or external store

Do not apply when: the feature is a background job, an event handler with no HTTP interface, a pure read-through cache, or a webhook receiver without resource state.

---

## Canonical Component Graph

```
C1 — Controller
  │  Owns: HTTP request/response, input validation, route handling
  │  Does not own: business logic, data access
  │
  ▼
C2 — Service
  │  Owns: business logic, orchestration, domain rules
  │  Does not own: HTTP concerns, raw query building
  │
  ▼
C3 — Repository
     Owns: data access, query building, persistence abstraction
     Does not own: business logic, HTTP concerns
```

Dependencies flow downward only. C1 depends on C2. C2 depends on C3. C3 depends on nothing in the feature.

---

## Required Interfaces (abstract)

These are the shapes each component must expose. Concrete signatures are resolved in the bootstrap.

**C1 — Controller**
- Route registration method (or decorator) that binds HTTP verbs to handler methods
- Handler methods that accept a request context and return a response
- Input validation before delegating to the service

**C2 — Service**
- One method per business operation (create, findById, update, delete, list…)
- Accepts domain input types (not raw HTTP request objects)
- Returns domain objects or throws typed errors
- No awareness of HTTP status codes or response formatting

**C3 — Repository**
- One method per data operation
- Accepts and returns domain types (not raw ORM models unless the project maps them)
- Abstracts the query builder or ORM — callers do not import it directly

---

## Adaptation Questions

The bootstrap generator must answer these by surveying the codebase.

1. What is the base class or interface for controllers/route handlers? Where is it defined?
2. How are routes registered? (decorator, router object, central registry, framework convention)
3. What validation library is used? Where are schemas typically defined?
4. What ORM or query builder is used? What is the base repository class or pattern?
5. How are services instantiated and injected into controllers? (constructor injection, DI container, factory)
6. What error types are thrown from services? How do controllers convert them to HTTP responses?
7. What does a success response look like? (direct object, envelope `{ data: ... }`, other)
8. What does a validation failure response look like?
9. Where are new routes registered? (a routes index file, auto-discovery, framework convention)
10. Where are new services and repositories registered for dependency injection (if applicable)?

---

## Known Anti-Patterns

These must appear in the bootstrap's Constraints section, confirmed against the actual codebase.

- **Controller bloat:** business logic (conditionals, calculations, domain rules) placed in the controller. All logic belongs in the service.
- **Layer skipping:** controller or service calls the ORM/database directly, bypassing the repository. Data access belongs in the repository.
- **HTTP leakage:** service methods accept raw request objects or return HTTP status codes. Services are HTTP-unaware.
- **Fat repository:** repository methods encode business rules or make decisions beyond data access. Repositories fetch and persist; they do not decide.
- **Missing validation:** service receives unvalidated input from the controller. Validation must happen in the controller before delegation.
- **Implicit error handling:** errors swallowed or converted silently without logging. All unexpected errors must be logged before re-throwing or converting.
