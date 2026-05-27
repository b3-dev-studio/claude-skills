---
artifact: spec
version: 1
gate_status: pending
open_questions_count: 0
---

# Spec: {{feature_name}}

## Functional Requirements

### F1 — {{requirement_name}}
{{One sentence: what the system must do.}}

**Acceptance Criteria**
- F1.AC1: Given {{context}} / When {{action}} / Then {{outcome}}
- F1.AC2: Given {{context}} / When {{action}} / Then {{outcome}}

### F2 — {{requirement_name}}
{{One sentence: what the system must do.}}

**Acceptance Criteria**
- F2.AC1: Given {{context}} / When {{action}} / Then {{outcome}}

## Non-Functional Requirements

### NF1 — {{nfr_name}}
{{Description.}} *(measurable: yes — {{metric, e.g. "< 3s p95 latency"}})*

### NF2 — {{nfr_name}}
{{Description.}} *(measurable: yes — {{metric}})*

## Out of Scope
- {{Explicit exclusion — what this feature will NOT do}}
- {{Explicit exclusion}}

## Open Questions
*None. This section must be empty before the gate can pass.*

---

## Gate: Spec → Architecture

**Validator checks (all must pass):**
- [ ] `open_questions_count` in frontmatter equals 0
- [ ] Every NFR marked `measurable: yes` with a concrete metric
- [ ] Every AC follows `Given / When / Then` structure
- [ ] At least one functional requirement exists
- [ ] Out of Scope section contains at least one entry
