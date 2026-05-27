---
artifact: arch
version: 1
gate_status: pending
satisfies: [F1, F2, NF1, NF2]   # must list every requirement ID from spec
---

# Architecture: {{feature_name}}

## Requirement Coverage
| Requirement | Component(s) | Notes |
|-------------|--------------|-------|
| F1          | C1, C2       |       |
| F2          | C2           |       |
| NF1         | C1           | {{e.g. "connection pooling strategy"}} |
| NF2         | C3           |       |

## Components

### C1 — {{ComponentName}}
**File:** `{{src/path/to/file.ts}}`
**Satisfies:** F1, NF1
**Responsibility:** {{One sentence describing what this component owns.}}

**Interfaces**
```
{{methodName}}({{param}}: {{Type}}): {{ReturnType}}
{{methodName}}({{param}}: {{Type}}): {{ReturnType}}
```

**Dependencies**
- Internal: C2
- Existing: `{{ExistingService}}` at `{{src/path/to/existing.ts}}`

---

### C2 — {{ComponentName}}
**File:** `{{src/path/to/file.ts}}`
**Satisfies:** F1, F2
**Responsibility:** {{One sentence.}}

**Interfaces**
```
{{methodName}}({{param}}: {{Type}}): {{ReturnType}}
```

**Dependencies**
- Internal: none
- Existing: none

---

### C3 — {{ComponentName}}
**File:** `{{src/path/to/file.ts}}`
**Satisfies:** NF2
**Responsibility:** {{One sentence.}}

**Interfaces**
```
{{methodName}}({{param}}: {{Type}}): {{ReturnType}}
```

**Dependencies**
- Internal: C1

---

## Integration Points
| Existing File | Change | Rationale |
|---------------|--------|-----------|
| `{{src/routes/existing.ts}}` | {{e.g. "add /new-route handler"}} | {{which requirement drives this}} |
| `{{src/config/index.ts}}`    | {{e.g. "add feature flag"}}      | {{which requirement drives this}} |

## Constraints on Implementation
- {{What is explicitly forbidden — e.g. "No new auth primitives outside C1"}}
- {{What is required — e.g. "All state parameters must be validated before use"}}
- {{Naming or structural convention to follow}}

## Rejected Approaches
- **{{Approach name}}**: rejected because {{reason tied to requirements or codebase context}}
- **{{Approach name}}**: rejected because {{reason}}

---

## Gate: Architecture → Contract

**Validator checks (all must pass):**
- [ ] Every requirement ID from spec's frontmatter appears in this doc's `satisfies`
- [ ] Every requirement ID appears in the Coverage table
- [ ] Every component has: file path, satisfies list, responsibility, interfaces, dependencies
- [ ] Every integration point has a rationale referencing a requirement ID
- [ ] Constraints on Implementation is non-empty
- [ ] Rejected Approaches documents at least one alternative considered
