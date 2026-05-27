---
artifact: bootstrap
blueprint: {{blueprint-name}}
project: {{project-name}}
generated: {{YYYY-MM-DD}}
gate_status: ready
---

# Bootstrap: {{Blueprint Display Name}} for {{project-name}}

## Source Blueprint
`{{path/to/blueprint.md}}` — v{{version}}

## Representative Example
The closest existing feature of this type in the codebase:

**Feature:** {{name of the existing feature, e.g. "User registration endpoint"}}
**Entry point:** `{{src/path/to/entry.ts}}:{{line}}`
**Trace summary:** {{one paragraph tracing the feature from entry to data layer}}

---

## Resolved Component Graph

### C1 — {{Role from blueprint, e.g. "Controller"}}
**Base class / interface:** `{{ClassName}}` at `{{src/path/to/base.ts}}:{{line}}`
**Naming convention:** `{{Resource}}{{Role}}` (e.g., `UserController`)
**File location:** `{{src/path/to/controllers/}}`
**Responsibility:** {{one sentence — what this layer owns in this codebase}}

**Concrete interfaces** (from the representative example):
```ts
// {{src/path/to/example.ts}}:{{line}}
{{actual method signatures from the example}}
```

---

### C2 — {{Role from blueprint, e.g. "Service"}}
**Base class / interface:** `{{ClassName}}` at `{{src/path/to/base.ts}}:{{line}}` *(or "none — standalone class")*
**Naming convention:** `{{Resource}}{{Role}}`
**File location:** `{{src/path/to/services/}}`
**Responsibility:** {{one sentence}}

**Concrete interfaces:**
```ts
// {{src/path/to/example.ts}}:{{line}}
{{actual method signatures from the example}}
```

---

### C3 — {{Role from blueprint, e.g. "Repository"}}
**Base class / interface:** `{{ClassName}}` at `{{src/path/to/base.ts}}:{{line}}` *(or "none")*
**Naming convention:** `{{Resource}}{{Role}}`
**File location:** `{{src/path/to/repositories/}}`
**Responsibility:** {{one sentence}}

**Concrete interfaces:**
```ts
// {{src/path/to/example.ts}}:{{line}}
{{actual method signatures from the example}}
```

---

## Adaptation Answers

Answers to the blueprint's adaptation questions, with evidence.

| Question | Answer | Evidence |
|----------|--------|----------|
| {{adaptation question 1}} | {{answer}} | `{{file}}:{{line}}` |
| {{adaptation question 2}} | {{answer}} | `{{file}}:{{line}}` |
| {{adaptation question 3}} | {{answer}} | `{{file}}:{{line}}` |

---

## Behavioral Patterns

These are loaded as the cross-cutting brief during implementation. Every file in a feature of this type must follow these conventions.

### Error Handling
{{How errors are created and thrown. Which error classes are used. How they propagate.}}

```ts
// Example from {{src/path/to/example.ts}}:{{line}}
{{actual error handling code excerpt}}
```

### Input Validation
{{Where validation happens (controller vs middleware). What library. What a validation failure returns.}}

```ts
// Example from {{src/path/to/example.ts}}:{{line}}
{{actual validation code excerpt}}
```

### Response Shape
{{The envelope or direct structure used for success responses. What error responses look like.}}

```ts
// Success: {{shape description}}
// Error:   {{shape description}}
// Example from {{src/path/to/example.ts}}:{{line}}
{{actual response code excerpt}}
```

### Logging
{{What gets logged. At what level. Using what logger. Format.}}

### Transactions
{{How multi-step data operations are wrapped. ORM transaction pattern if applicable.}}
*(or "Not applicable for this pattern")*

---

## Integration Points

Existing files that features of this type routinely touch, beyond the component files themselves.

| File | Purpose | How it's modified |
|------|---------|-------------------|
| `{{src/path/to/routes.ts}}` | Route registration | Add route entry |
| `{{src/path/to/di-container.ts}}` | Dependency injection | Register new service/repo |
| `{{src/path/to/index.ts}}` | Barrel export | Export new class |

---

## Constraints

Inherited from the blueprint's known anti-patterns, confirmed against this codebase.

- {{Constraint derived from anti-pattern — e.g. "No business logic in controllers. The representative example confirms all logic lives in the service layer."}}
- {{Constraint}}
- {{Constraint}}

---

## Codebase Notes

Inconsistencies or deviations from the blueprint's canonical form found in this codebase.

- {{Note — e.g. "Two older features bypass the repository layer and call Prisma directly. These are legacy — new features must use the repository pattern."}}
*(or "None — codebase is consistent with the blueprint.")*
