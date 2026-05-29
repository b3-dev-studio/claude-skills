---
name: sdlc
description: "Full autonomous SDLC workflow. Use when the user wants to build a new feature end-to-end: from requirements through architecture, implementation, and verification. Each phase produces a structured artifact that gates entry to the next — no phase begins until the previous artifact passes its validator."
argument-hint: <feature description>
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, TodoWrite, Agent]
---

# Autonomous SDLC

Drive a feature from requirements to verified implementation through a cascade of constrained artifacts. Each phase produces a structured markdown document. A validator checks the document against a gate checklist before the next phase begins. The model's degrees of freedom narrow at each step: the spec constrains the architecture, the architecture constrains the contract, the contract constrains the implementation.

---

## Standing Context

Load these once at the start. Hold them in working memory. Do not re-read them per phase.

| Source | What it provides |
|--------|-----------------|
| `CLAUDE.md` | Project conventions, library choices, commands, module structure |
| `.sdlc/decision-log.md` | Architectural decisions made across prior features (load if exists) |
| `.sdlc/vocabulary.md` | Canonical domain terms (load if exists) |

---

## Artifact Locations

All run artifacts live in `.sdlc/current/`. Create the directory if it does not exist.

```
.sdlc/
├── current/
│   ├── spec.md
│   ├── arch.md
│   ├── contract.md
│   └── verification.md
├── decision-log.md
├── vocabulary.md
├── blueprints/        ← codebase-agnostic pattern templates
└── bootstraps/        ← codebase-specific pattern surveys (cached)
```

---

## Phase 1: Spec

**Goal:** Produce a fully specified, unambiguous requirements document.

**Context:** Standing context only.

**Input detection — run this first:**

Check whether `$ARGUMENTS` is a path to an approved Product Brief:
- If `$ARGUMENTS` ends in `.md` and the path contains `briefs/` → **Brief path** (see below)
- Otherwise → **Free-text path** (see below)

---

### Brief Path — bootstrapping from a Product Brief

1. Read the file at `$ARGUMENTS`.
2. Check the frontmatter field `status`. If it is not `approved`, halt immediately and tell the user:
   > "This brief has status `<status>`. Only briefs with `status: approved` can be used as SDLC input. Edit the brief's frontmatter to set `status: approved` when ready."
3. Extract these fields from the brief to seed the spec:
   - `## Opportunity Statement` → one-sentence feature description (becomes the spec preamble)
   - `## Success Criteria` → acceptance criteria; convert each numbered criterion to `Given / When / Then` form
   - `## Scope / In Scope` → drives the functional requirement boundaries
   - `## Scope / Out of Scope` → paste directly into spec's Out of Scope section
   - `## PRODUCT.md Goals Served` → check `PRODUCT.md` for any quantitative metrics on those goals; use them as NFR candidates
   - `suggested-blueprint` frontmatter → hold for Phase 2 (do not discard)
4. Create a TodoWrite list covering all phases.
5. Check `.sdlc/vocabulary.md`. All requirement statements must use canonical terms.
6. Write `.sdlc/current/spec.md` using the spec template, pre-populated from the brief. Open Questions must be empty — the brief approval is the requirements sign-off.
7. Present the spec to the user with a note that it was bootstrapped from the brief. **Wait for explicit approval before running the gate.**

---

### Free-text Path — bootstrapping from a feature description

1. Create a TodoWrite list covering all phases.
2. Parse `$ARGUMENTS`. If the feature is vague, ask:
   - What problem does this solve?
   - Who uses it and how?
   - What are the hard constraints (performance, security, backward compatibility)?
3. Check `.sdlc/vocabulary.md`. All requirement statements must use canonical terms. For any term not in the vocabulary, confirm the canonical name with the user before writing the spec.
4. Write `.sdlc/current/spec.md` using the spec template:
   - Each functional requirement gets a unique ID: F1, F2…
   - Each acceptance criterion gets a compound ID: F1.AC1, F1.AC2…
   - Each NFR gets an ID: NF1, NF2… — and a concrete measurable metric
   - Out of Scope must contain at least one explicit exclusion
   - Open Questions must be empty
5. Present the spec to the user. **Wait for explicit approval before running the gate.**

---

**Gate — Spec → Architecture:**
Run the artifact-validator agent:
```
artifact: .sdlc/current/spec.md
checks: spec gate checklist (see spec template)
```
On violations: fix and re-run. Do not proceed until `gate_status: passed`.

**Scope lock:** Once this gate passes, the spec is frozen. Any downstream phase that needs to add or remove scope must emit a Scope Change Request and return here.

---

## Phase 2: Blueprint Selection

**Goal:** Load the reusable architectural pattern that applies to this feature, or generate one.

**Context:** Standing context + `spec.md`.

**Actions:**
1. Determine the feature type:
   - If Phase 1 used the **Brief path**: use the `suggested-blueprint` field from the brief as the feature type. Treat it as the starting candidate — verify it still fits after reading `spec.md`.
   - Otherwise: infer the feature type from `spec.md`. Common types: `rest-api-resource`, `background-worker`, `oauth-integration`, `event-driven-handler`, `cache-layer`, `claude-code-plugin`.
2. Check `.sdlc/bootstraps/` for a matching bootstrap (filename = feature type).
   - **Bootstrap found:** Load it. Skip to Phase 3. No exploration needed.
   - **No bootstrap:** Check `.sdlc/blueprints/` for a matching blueprint.
     - **Blueprint found:** Survey the codebase for the blueprint's adaptation questions. Write the result to `.sdlc/bootstraps/{{feature-type}}.md`. This is a one-time cost — all future features of this type skip it.
     - **No blueprint:** Proceed to Phase 3 without a bootstrap. Note this explicitly in the arch doc.

---

## Phase 3: Architecture

**Goal:** Produce a component graph and interface contracts that fully cover every requirement in the spec.

**Context:** Standing context + `spec.md` + bootstrap (if exists).

**Actions:**
1. Launch the arch-designer agent with:
   - `spec.md`
   - The bootstrap or blueprint (if available)
   - The decision log
   - Instruction: *any deviation from the bootstrap's canonical component graph must appear under Rejected Approaches with rationale*
2. Read all key files the agent identifies before writing the arch doc.
3. Cross-check every component interface against the decision log. A component that contradicts a standing decision is a blocker — surface it to the user before proceeding.
4. Write `.sdlc/current/arch.md` using the arch template.
5. Present to the user. **Wait for explicit approval.**

**Gate — Architecture → Contract:**
Run the artifact-validator agent:
```
artifact: .sdlc/current/arch.md
cross-reference: .sdlc/current/spec.md
checks: arch gate checklist (see arch template)
```
On violations: fix and re-run. Do not proceed until `gate_status: passed`.

---

## Phase 4: Contract

**Goal:** Translate the architecture into a file-level implementation contract: exact export signatures and a test plan that maps every acceptance criterion to at least one test case.

**Context:** `arch.md` + spec acceptance criteria section only (not the full spec).

**Actions:**
1. For each component in `arch.md`, write a file contract entry with:
   - Required export signatures — must match the arch's interface definitions exactly
   - For modify actions: list unchanged exports explicitly
2. Map every acceptance criterion ID to at least one test case in the Test Plan table.
3. Populate Forbidden from the arch's constraints and the decision log.
4. Write `.sdlc/current/contract.md` using the contract template.
5. Present to the user. **Wait for explicit approval.**

**Gate — Contract → Implementation:**
Run the artifact-validator agent:
```
artifact: .sdlc/current/contract.md
cross-reference: .sdlc/current/arch.md, .sdlc/current/spec.md (ACs only)
checks: contract gate checklist (see contract template)
```
On violations: fix and re-run. Do not proceed until `gate_status: passed`.

---

## Phase 5: Implementation

**Goal:** Write code that conforms exactly to the contract, with consistent behavior across all files.

**DO NOT START WITHOUT EXPLICIT USER APPROVAL.**

**Setup — run once before writing any file:**
1. Extract a **cross-cutting brief** from `CLAUDE.md` and the bootstrap covering behavioral patterns only: error handling, logging, transaction boundaries, null/undefined treatment, naming conventions at the implementation level. Hold this for the entire phase — do not re-read `CLAUDE.md` per file.
2. Determine the **build sequence** from `arch.md`'s dependency graph: implement leaves first (repositories, utilities), then services, then controllers/handlers. File contracts that have no dependents go last.
3. Initialise an empty **running summary** list.

**Per file:**

*Context to load:*
- This file's contract entry
- Cross-cutting brief (already in working memory)
- All files written so far in this feature (small and directly relevant)
- Running summary list
- The specific existing files this contract entry touches (from the Integration Points in `arch.md`)

*Actions:*
1. Write the file following the contract entry's required signatures and the cross-cutting brief.
2. Run incremental validation:
   - Do exported names and signatures match the contract entry exactly?
   - For modify actions: are the listed unchanged exports unaltered?
   - If mismatch → fix before moving to the next file.
3. Append a one-line entry to the running summary:
   `{{ComponentID}} {{FileName}} — {{key behaviors and error conventions in one sentence}}`
4. If any implementation need falls outside the contract:
   - **Do not silently expand scope.**
   - Emit a Scope Change Request and pause until resolved.
5. Update TodoWrite as each file is completed.

---

## Phase 6: Verification

**Goal:** Confirm every acceptance criterion passes and the implementation conforms to the contract.

**Context:** `contract.md` + spec acceptance criteria only + test output.

**Actions:**
1. Run the test suite. Record pass/fail per test ID.
2. For each NFR, capture the measured value against its metric.
3. For each component, confirm exported signatures match the contract.
4. Write `.sdlc/current/verification.md` using the verification template.
5. Document every deviation from the contract with explicit justification. Undocumented deviations block the gate.

**Gate — Verification → Done:**
Run the artifact-validator agent:
```
artifact: .sdlc/current/verification.md
cross-reference: .sdlc/current/contract.md, .sdlc/current/spec.md (ACs only)
checks: verification gate checklist (see verification template)
```
On failures: diagnose, fix, re-run tests, re-run gate. Do not mark done until `gate_status: passed`.

---

## Phase 7: Close Out

**Goal:** Update persistent project memory and summarise.

**Actions:**
1. Append to `.sdlc/decision-log.md`:
   - Any architectural decisions made during this feature, with rationale
   - Any approaches explicitly rejected, and why
2. Append to `.sdlc/vocabulary.md` any new canonical terms introduced.
3. Confirm the bootstrap (if newly generated) is saved to `.sdlc/bootstraps/`.
4. Mark all TodoWrite items complete.
5. Summarise: what was built, key decisions, files modified, suggested next steps.

---

## Scope Change Request

Emitted by any phase when a need outside the frozen spec is discovered.

```markdown
## Scope Change Request
**Raised by:** {{phase name}}
**Discovered:** {{what was found that the spec does not cover}}
**Options:**
  a) Add to spec as {{new requirement ID}}
  b) Mark explicitly out of scope
  c) Defer to a follow-on feature
**Blocks:** {{current phase}} until resolved
```

Present to user. Wait for decision.
- If **(a)**: update `spec.md`, re-run the spec gate, then return to the phase that raised the request.
- If **(b)** or **(c)**: add the item to Out of Scope in `spec.md` and continue.

---

## Artifact Validator

Used at every gate. Invoke as the artifact-validator agent with the artifact path, any cross-references, and the gate checklist from the relevant template.

The validator:
- Reads the artifact's frontmatter and the gate checklist section
- Returns `{ passed: true }` or `{ passed: false, violations: [{check, what_failed, minimum_fix}] }`
- On pass: updates `gate_status: passed` in the artifact's frontmatter

The validator never reads the full codebase. It reads only the artifacts it is handed.
