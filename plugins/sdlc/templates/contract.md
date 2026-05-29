---
artifact: contract
version: 1
gate_status: pending
satisfies: [C1, C2, C3]   # must list every component ID from arch
---

# Implementation Contract: {{feature_name}}

## Component Coverage
| Component | File | Action |
|-----------|------|--------|
| C1 | `{{src/path/to/file.ts}}`      | create |
| C2 | `{{src/path/to/other.ts}}`     | create |
| C3 | `{{src/path/to/existing.ts}}`  | modify |

---

## File Contracts

### `{{src/path/to/file.ts}}` — create
**Implements:** C1

**Required exports — implementation must match these signatures exactly:**
```ts
export class {{ClassName}} {
  {{methodName}}({{param}}: {{Type}}): {{ReturnType}}
  {{methodName}}({{param}}: {{Type}}): {{ReturnType}}
}
```

**Test file:** `{{src/path/to/file.test.ts}}`

---

### `{{src/path/to/other.ts}}` — create
**Implements:** C2

**Required exports:**
```ts
export function {{functionName}}({{param}}: {{Type}}): {{ReturnType}}
export type {{TypeName}} = {
  {{field}}: {{Type}}
  {{field}}: {{Type}}
}
```

**Test file:** `{{src/path/to/other.test.ts}}`

---

### `{{src/path/to/existing.ts}}` — modify
**Implements:** C3
**Change:** {{Precise description of what is added or changed — not "update the file", but "add X method to Y class".}}

**New exports:**
```ts
export function {{newMethod}}({{param}}: {{Type}}): {{ReturnType}}
```

**Unchanged exports** *(must not be altered):*
- `{{existingFunction}}`
- `{{ExistingClass}}`

---

## Test Plan
| Test ID | Covers    | File | Description |
|---------|-----------|------|-------------|
| T1      | F1.AC1    | `{{file.test.ts}}` | {{Arrange: {{preconditions}}. Act: call {{function/method}} with {{inputs}}. Assert: {{expected return value or side effect and its type}}.}} |
| T2      | F1.AC2    | `{{file.test.ts}}` | {{Arrange: {{preconditions}}. Act: call {{function/method}} with {{inputs}}. Assert: {{expected return value or side effect and its type}}.}} |
| T3      | F2.AC1    | `{{file.test.ts}}` | {{Arrange: {{preconditions}}. Act: call {{function/method}} with {{inputs}}. Assert: {{expected return value or side effect and its type}}.}} |

*Every acceptance criteria ID from the spec must appear in the Covers column. Descriptions must be specific enough to write assertions from — "verifies it works" is not acceptable.*

## Forbidden
- {{What must not be created, modified, or introduced}}
- {{What pattern or dependency is off-limits}}
- {{What existing interface must not change signature}}

---

## Gate: Contract → Implementation

**Validator checks (all must pass):**
- [ ] Every component ID from arch's frontmatter appears in this doc's `satisfies`
- [ ] Every component appears in the Coverage table
- [ ] Every file contract has explicit required signatures
- [ ] For modify actions: unchanged exports are listed
- [ ] Every acceptance criteria ID from spec has ≥1 row in the Test Plan
- [ ] Forbidden list is non-empty
