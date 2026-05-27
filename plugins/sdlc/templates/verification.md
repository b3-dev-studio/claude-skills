---
artifact: verification
version: 1
gate_status: pending
all_criteria_met: false
unaddressed_criteria: []
---

# Verification Report: {{feature_name}}

## Acceptance Criteria Results
| Criteria ID | Test | Status | Evidence |
|-------------|------|--------|----------|
| F1.AC1 | T1 | {{pass/fail}} | `{{file.test.ts}}:{{line}}` |
| F1.AC2 | T2 | {{pass/fail}} | `{{file.test.ts}}:{{line}}` |
| F2.AC1 | T3 | {{pass/fail}} | `{{file.test.ts}}:{{line}}` |

*Every acceptance criteria ID from spec must have a row. Any missing → blocked.*

## NFR Results
| NFR ID | Metric | Measured | Status |
|--------|--------|----------|--------|
| NF1    | {{e.g. "< 3s p95"}} | {{actual measurement}} | {{pass/fail}} |
| NF2    | {{metric}}          | {{actual measurement}} | {{pass/fail}} |

*Measurement method: {{e.g. "integration test with 100 iterations", "load test at 50rps"}}*

## Contract Conformance
| Component | File | Signatures Match | Test Coverage |
|-----------|------|-----------------|---------------|
| C1 | `{{src/path/to/file.ts}}`  | {{yes/no}} | {{yes/no}} |
| C2 | `{{src/path/to/other.ts}}` | {{yes/no}} | {{yes/no}} |
| C3 | `{{src/path/to/existing.ts}}` | {{yes/no}} | {{yes/no}} |

## Deviations from Contract
*Any deviation from the contract requires explicit justification here.
Undocumented deviations block gate passage.*

| Component | Deviation | Justification | Approved |
|-----------|-----------|---------------|---------|
| — | — | — | — |

## Forbidden Violations
*List any forbidden items from the contract that were violated, with justification.
An unjustified violation blocks gate passage.*

| Forbidden Item | Violated | Justification |
|----------------|----------|---------------|
| — | no | — |

---

## Gate: Verification → Done

**Validator checks (all must pass):**
- [ ] Every acceptance criteria ID from spec has a result row
- [ ] Every NFR has a concrete measured value
- [ ] Every contract component has a conformance row
- [ ] No `fail` in Acceptance Criteria Results
- [ ] No `fail` in NFR Results
- [ ] No `no` in Signatures Match without a justified deviation entry
- [ ] `all_criteria_met: true` in frontmatter
- [ ] `unaddressed_criteria: []` in frontmatter
- [ ] All deviations have justifications
- [ ] All forbidden violations have justifications
