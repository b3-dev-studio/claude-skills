---
artifact: spec
version: 1
gate_status: passed
open_questions_count: 0
---

# Spec: PM Skill (Product Management Plugin)

## Resolved Design Decisions

The following questions from `plans/pm-skill.md` were resolved before writing this spec:

1. **Scheduling** — On-demand only. Users who want periodic runs use `/schedule` separately.
2. **Competitive scan scope** — Whole product by default. An optional argument (`/pm focus=<area>`) narrows to a feature area.
3. **Moonshot quality** — Each moonshot idea must cite ≥ 1 PRODUCT.md goal it serves. Ideas that cannot be grounded in a stated goal are discarded.
4. **PRODUCT.md bootstrapping** — The skill auto-generates a starter PRODUCT.md from codebase analysis if none exists, then presents it to the user for review before proceeding.
5. **Brief prioritization** — The opportunity-synthesizer ranks briefs by estimated impact × effort inverse. The human reorders the queue manually before approving.

---

## Functional Requirements

### F1 — PRODUCT.md Bootstrap
If `PRODUCT.md` does not exist in the project root, the skill must analyze the codebase and generate a starter `PRODUCT.md` covering: product vision, success metrics, target users, strategic priorities, known constraints, and explicit non-goals. The generated file is presented to the user for review before the skill proceeds.

**Acceptance Criteria**
- F1.AC1: Given no `PRODUCT.md` exists / When the PM skill runs / Then a starter `PRODUCT.md` is written to the project root and the user is shown it before any further analysis begins.
- F1.AC2: Given `PRODUCT.md` exists / When the PM skill runs / Then the existing file is read and no bootstrap is performed.
- F1.AC3: Given a generated `PRODUCT.md` / When the user reviews it / Then it contains all six required sections (vision, metrics, users, priorities, constraints, non-goals).

### F2 — Product Audit
The skill must read the local codebase, `PRODUCT.md`, and `.sdlc/decision-log.md` (if present) to produce a structured assessment of the current product state, including: feature inventory, what is working well, what is incomplete or missing, and gaps between stated goals and implemented reality.

**Acceptance Criteria**
- F2.AC1: Given a codebase and `PRODUCT.md` / When the audit runs / Then a feature inventory is produced listing implemented capabilities.
- F2.AC2: Given `PRODUCT.md` goals and the feature inventory / When the audit runs / Then each PRODUCT.md goal is mapped to its implementation status (implemented / partial / missing).
- F2.AC3: Given `.sdlc/decision-log.md` exists / When the audit runs / Then the decision log is incorporated into the assessment of what is working well or has known issues.
- F2.AC4: Given the audit completes / When results are produced / Then no web search is used (local-only analysis).

### F3 — Competitive Scan
The skill must use web search to identify 3–5 comparable products or tools in the same domain, map capabilities they have that the current product lacks, identify current product differentiators, and surface trends in the space.

**Acceptance Criteria**
- F3.AC1: Given the product domain from `PRODUCT.md` and the feature inventory from F2 / When the competitive scan runs / Then 3–5 comparable products are identified and documented.
- F3.AC2: Given the comparable products / When the competitive scan runs / Then for each comparable, at least one capability gap (what they have, current product lacks) is documented.
- F3.AC3: Given the competitive scan results / When results are produced / Then at least one current product differentiator is identified.
- F3.AC4: Given an optional `focus=<area>` argument / When the PM skill runs / Then the competitive scan is scoped to the specified feature area rather than the whole product.

### F4 — Opportunity Synthesis
The skill must cross-reference the audit (F2) and competitive scan (F3) with `PRODUCT.md` goals to produce a ranked list of opportunities categorized as: gap opportunities, improvement opportunities, and moonshot opportunities. Each moonshot must cite ≥ 1 PRODUCT.md goal it serves.

**Acceptance Criteria**
- F4.AC1: Given audit + competitive scan results + PRODUCT.md / When synthesis runs / Then opportunities are categorized into gap, improvement, and moonshot types.
- F4.AC2: Given ranked opportunities / When synthesis completes / Then each opportunity has an estimated impact (high/medium/low) and effort (high/medium/low).
- F4.AC3: Given moonshot opportunities / When synthesis completes / Then each moonshot that cites ≥ 1 PRODUCT.md goal it serves is included in the ranked list; moonshots with no valid goal citation are placed in a separate `discarded-moonshots` section with the reason stated.
- F4.AC4: Given ranked opportunities / When presented to the user / Then opportunities are ordered by impact-to-effort ratio (highest first).
- F4.AC5: Given a `discarded-moonshots` section / When presented to the user / Then each entry shows the idea title, its origin (synthesized or human-sourced), and the reason it was discarded (e.g., "no PRODUCT.md goal citation found").

### F5 — Brief Generation
The skill must produce a Product Brief markdown file in `.sdlc/briefs/` for each opportunity the user selects. Each brief must be structured to serve as a SDLC Phase 1 input: containing an opportunity statement, evidence, success criteria (as testable outcomes), scope boundaries, and a suggested SDLC blueprint type.

**Acceptance Criteria**
- F5.AC1: Given a user-selected opportunity / When a brief is generated / Then a file is written to `.sdlc/briefs/<slug>.md` with frontmatter `status: pending`.
- F5.AC2: Given a Product Brief / When read by SDLC Phase 1 / Then it contains: opportunity statement, evidence, success criteria, scope-in, scope-out, and suggested-blueprint fields.
- F5.AC3: Given success criteria in the brief / When written / Then each criterion is expressed as a testable outcome (measurable or binary).
- F5.AC4: Given multiple opportunities approved for briefs / When briefs are generated / Then each brief is a separate file with a distinct slug derived from the opportunity title.

### F7 — Human Idea Injection
The skill must check for `.sdlc/pm-ideas.md` at the start of each run. If it exists, unchecked items (`- [ ]`) are fed into the opportunity synthesizer alongside synthesized opportunities. After evaluation, the skill marks each evaluated idea as checked (`- [x]`) in the file. Checked items are skipped on subsequent runs to avoid re-evaluation cost. The human can uncheck an idea to force re-evaluation.

**Acceptance Criteria**
- F7.AC1: Given `.sdlc/pm-ideas.md` exists with unchecked items / When synthesis runs / Then unchecked ideas are evaluated and ranked alongside synthesized opportunities, each flagged as `human-sourced`.
- F7.AC2: Given `.sdlc/pm-ideas.md` exists with only checked items / When synthesis runs / Then no human ideas are sent to the synthesizer (zero additional token cost).
- F7.AC3: Given unchecked ideas that were evaluated in this run / When the run completes / Then each evaluated idea is marked `- [x]` in `.sdlc/pm-ideas.md`.
- F7.AC4: Given `.sdlc/pm-ideas.md` does not exist / When synthesis runs / Then the skill proceeds normally with no error and no human-sourced ideas.
- F7.AC5: Given a checked idea that the human has manually unchecked / When synthesis runs / Then it is re-evaluated as if new.

### F6 — PM Log Update
After every complete PM skill run, the skill must append a cycle summary to `.sdlc/pm-log.md` recording: run date, product domain, audit findings summary, competitive scan summary, opportunities identified, and briefs generated.

**Acceptance Criteria**
- F6.AC1: Given a completed PM run / When the run ends / Then a cycle summary is appended to `.sdlc/pm-log.md` (created if not exists).
- F6.AC2: Given `.sdlc/pm-log.md` exists from prior runs / When a new cycle summary is appended / Then prior entries are not modified.
- F6.AC3: Given a cycle summary / When written / Then it records: ISO date, product domain, count of opportunities found, count of briefs generated, and a one-paragraph findings summary.

---

## Non-Functional Requirements

### NF1 — Competitive Scan Cost
The competitive scan must not perform more than 10 web searches per PM run. *(measurable: yes — ≤ 10 WebSearch + WebFetch calls total during F3)*

### NF2 — Brief SDLC Compatibility
Every Product Brief produced by the PM skill must be parseable by SDLC Phase 1 without modification — the brief's success criteria map directly to SDLC spec acceptance criteria, and the suggested-blueprint field maps to a known blueprint name. *(measurable: yes — SDLC Phase 1 can produce a spec.md from the brief with zero clarifying questions)*

### NF3 — No External State Mutation
The PM skill must not modify any SDLC artifact outside `.sdlc/briefs/`, `.sdlc/pm-log.md`, and `.sdlc/pm-ideas.md` (checkbox updates only). In particular, it must not write to `.sdlc/current/`, `.sdlc/decision-log.md`, or any source file. *(measurable: yes — no file writes outside the three permitted paths and PRODUCT.md bootstrap)*

---

## Out of Scope

- **Auto-scheduling**: The PM skill does not set up its own cron schedule. Users who want periodic runs use the `/schedule` skill separately.
- **Automated brief approval**: No brief is marked `approved` without explicit human confirmation. The PM skill writes briefs as `status: pending` and stops.
- **SDLC execution**: The PM skill does not invoke `/sdlc`. It produces briefs; the human triggers SDLC separately.
- **Interactive PRODUCT.md editing**: After bootstrap, the PM skill reads PRODUCT.md but does not offer to edit it during a run. Editing is the user's responsibility.
- **Brief queue reordering**: The PM skill writes briefs in synthesis-ranked order. The human reorders the queue manually before approving.
- **Multi-product support**: A single run covers one product (one PRODUCT.md). Running against multiple codebases simultaneously is out of scope.

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
