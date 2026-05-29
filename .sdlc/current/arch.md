---
artifact: arch
version: 1
gate_status: passed
satisfies: [F1, F2, F3, F4, F5, F6, F7, NF1, NF2, NF3]
---

# Architecture: PM Skill (Product Management Plugin)

## No Bootstrap Available

No `claude-code-plugin` blueprint exists in `.sdlc/blueprints/` or `.sdlc/bootstraps/`. This architecture was derived by surveying the SDLC plugin at `plugins/sdlc/` as the closest existing feature. The SDLC plugin pattern uses: a SKILL.md orchestrator (entry point with frontmatter declaring name, tools, and argument-hint), agent markdown files (sub-agents invoked by the orchestrator), template markdown files (output scaffolds), and a `plugin.json` manifest. This PM skill follows that same structure exactly. This feature is a candidate for creating a `claude-code-plugin` blueprint once implemented.

---

## Requirement Coverage

| Requirement | Component(s) | Notes |
|-------------|--------------|-------|
| F1          | C1, C2       | C1 checks for PRODUCT.md and gates pipeline; C2 performs codebase analysis to generate starter PRODUCT.md |
| F2          | C2           | Product Auditor runs local-only analysis |
| F3          | C3           | Competitive Scanner performs web research |
| F4          | C4           | Opportunity Synthesizer cross-references all inputs and ranks |
| F5          | C5, C6       | Brief Writer fills C6 template and writes to .sdlc/briefs/ |
| F6          | C1           | Orchestrator appends cycle summary to .sdlc/pm-log.md after pipeline completes |
| F7          | C1, C4       | C1 reads and parses pm-ideas.md; C4 evaluates human ideas alongside synthesized ones; C1 marks evaluated ideas checked |
| NF1         | C3           | Competitive Scanner is constrained to ≤ 10 WebSearch + WebFetch calls per run |
| NF2         | C5, C6       | Brief Writer follows C6 template whose fields map directly to SDLC Phase 1 spec fields |
| NF3         | C1           | Orchestrator enforces write boundaries: only PRODUCT.md (bootstrap only), .sdlc/briefs/, .sdlc/pm-log.md, .sdlc/pm-ideas.md (checkbox updates) |

---

## Components

### C1 — PM Skill Orchestrator

**File:** `plugins/pm/skills/pm/SKILL.md`
**Satisfies:** F1, F6, F7, NF3
**Responsibility:** Entry point for the `/pm` skill that drives the four-agent sequential pipeline, enforces the PRODUCT.md bootstrap gate, manages human idea injection from pm-ideas.md, and appends the cycle summary to pm-log.md when the pipeline completes.

**Interfaces**

```
Input:
  $ARGUMENTS: optional string of the form "focus=<area>"
    - If present: the focus area is passed as a scoping parameter to C3 and C4
    - If absent: whole-product scope is used

Startup checks (executed before invoking any sub-agent):
  1. Read PRODUCT.md from the project root
     - If missing: invoke C2 in bootstrap mode, present output to user, pause for review
     - If present: load into working memory as product-context
  2. Read .sdlc/pm-ideas.md (if exists)
     - Extract all unchecked items (lines matching "- [ ] ...")
     - Pass as human-ideas list to C4; if empty/absent, pass empty list

Pipeline invocation sequence:
  Step 1: Invoke C2 (Product Auditor) → receive audit-report
  Step 2: Invoke C3 (Competitive Scanner) with (product-context, audit-report, focus?) → receive competitive-landscape
  Step 3: Invoke C4 (Opportunity Synthesizer) with (product-context, audit-report, competitive-landscape, human-ideas, focus?) → receive ranked-opportunities
  Step 4: Present ranked-opportunities to user; receive user-selected opportunities
  Step 5: For each selected opportunity: invoke C5 (Brief Writer) → .sdlc/briefs/<slug>.md written

Post-pipeline writes (all permitted by NF3):
  - .sdlc/pm-log.md: append cycle summary (created if absent)
  - .sdlc/pm-ideas.md: mark each evaluated idea as "- [x] ..." (checkbox update only)

Permitted write targets (NF3 enforcement):
  - PRODUCT.md: bootstrap mode only
  - .sdlc/briefs/<slug>.md: via C5 only
  - .sdlc/pm-log.md: append only
  - .sdlc/pm-ideas.md: checkbox updates only
  - All other files: read-only
```

**Dependencies**
- Internal: C2, C3, C4, C5
- Existing: `plugins/sdlc/skills/sdlc/SKILL.md` (structural pattern reference — not a runtime dependency)

---

### C2 — Product Auditor

**File:** `plugins/pm/agents/product-auditor.md`
**Satisfies:** F1, F2
**Responsibility:** Reads the local codebase, PRODUCT.md, and .sdlc/decision-log.md (if present) using only local file tools (no web search) to produce a structured audit report covering feature inventory, goal-to-implementation mapping, and gaps; when invoked in bootstrap mode, generates a starter PRODUCT.md from codebase analysis alone.

**Interfaces**

```
Invocation modes:

MODE A — Bootstrap (triggered when PRODUCT.md absent, satisfies F1):
  Input:
    - Project root directory path
  Output:
    - Structured PRODUCT.md draft with six required sections:
        ## Vision
        ## Success Metrics
        ## Target Users
        ## Strategic Priorities
        ## Known Constraints
        ## Non-Goals

MODE B — Audit (normal run, satisfies F2):
  Input:
    - product-context: PRODUCT.md content
    - Project root directory path
    - decision-log: .sdlc/decision-log.md content (if exists; omit if absent)
  Output (structured markdown):
    audit-report:
      feature-inventory: list of implemented capabilities
      goal-status: table of PRODUCT.md goal → [implemented | partial | missing]
      strengths: what is working well
      gaps: incomplete or missing items
      decision-log-notes: relevant entries (omitted if no decision-log)

Constraints:
  - No WebSearch or WebFetch calls (F2.AC4)
  - No file writes (NF3)
```

**Dependencies**
- Internal: none
- Existing: none (reads project files via Read/Glob/Grep)

---

### C3 — Competitive Scanner

**File:** `plugins/pm/agents/competitive-scanner.md`
**Satisfies:** F3, NF1
**Responsibility:** Uses web search (WebSearch and WebFetch, capped at 10 calls total per run) to identify 3–5 comparable products, map capability gaps, surface current product differentiators, and document emerging trends; narrows scope to a specified feature area when focus is provided.

**Interfaces**

```
Input:
  - product-context: PRODUCT.md content (domain, vision, feature summary)
  - audit-report: feature-inventory section from C2
  - focus: optional string — feature area to scope the scan

Process:
  - Execute WebSearch and WebFetch — total combined must not exceed 10
  - Identify 3–5 comparable products
  - For each comparable: document ≥ 1 capability gap
  - Identify ≥ 1 current product differentiator
  - Surface ≥ 1 trend

Output (structured markdown):
  competitive-landscape:
    comparables: list of 3–5 products, each with:
      name, url, capability-gaps (list)
    differentiators: list
    trends: list
    search-budget-used: integer (≤ 10)

Constraints:
  - WebSearch + WebFetch combined ≤ 10 (NF1)
  - 3–5 comparables required (F3.AC1)
  - No file writes (NF3)
```

**Dependencies**
- Internal: none
- Existing: none

---

### C4 — Opportunity Synthesizer

**File:** `plugins/pm/agents/opportunity-synthesizer.md`
**Satisfies:** F4, F7
**Responsibility:** Cross-references the audit report, competitive landscape, PRODUCT.md goals, and human-sourced ideas to produce a ranked list of opportunities categorized as gap, improvement, or moonshot, with moonshots required to cite at least one PRODUCT.md goal and all opportunities scored by impact-to-effort ratio.

**Interfaces**

```
Input:
  - product-context: PRODUCT.md content (goals section required)
  - audit-report: full C2 output
  - competitive-landscape: full C3 output
  - human-ideas: list of unchecked items from pm-ideas.md (may be empty)
  - focus: optional string

Process:
  - Derive gap opportunities from audit gaps + competitive capability-gaps
  - Derive improvement opportunities from feature-inventory + competitive comparisons
  - Generate moonshot opportunities from competitive trends; soft filter: moonshots that cite ≥ 1 PRODUCT.md goal enter the ranked list; moonshots with no valid goal citation are placed in discarded-moonshots with the reason stated
  - Evaluate human-ideas: classify, apply moonshot soft filter, score, rank alongside synthesized opportunities
  - Rank all retained opportunities by impact-to-effort ratio (highest first)

Output (structured markdown):
  ranked-opportunities:
    - rank, title, category (gap|improvement|moonshot), source (synthesized|human-sourced),
      impact (high|medium|low), effort (high|medium|low), summary, goal-citations, evidence
  discarded-moonshots:
    - title, source (synthesized|human-sourced), reason (e.g. "no PRODUCT.md goal citation found")
  evaluated-ideas: list of human-ideas that were evaluated this run (returned to C1 for checkbox marking)

Constraints:
  - Moonshots without ≥ 1 PRODUCT.md goal citation go to discarded-moonshots, not ranked-opportunities (F4.AC3, F4.AC5)
  - evaluated-ideas list must be returned for C1 to mark checkboxes (F7.AC3)
  - No web search or file writes (NF3)
```

**Dependencies**
- Internal: none
- Existing: none

---

### C5 — Brief Writer

**File:** `plugins/pm/agents/brief-writer.md`
**Satisfies:** F5, NF2
**Responsibility:** Takes a single ranked opportunity from C4 and fills the C6 product brief template to produce a complete Product Brief file at `.sdlc/briefs/<slug>.md` with `status: pending`, structured for direct use as SDLC Phase 1 input.

**Interfaces**

```
Input:
  - opportunity: one entry from C4 ranked-opportunities (single invocation per brief)
  - product-context: PRODUCT.md content
  - audit-report: C2 output (for evidence)
  - competitive-landscape: C3 output (for evidence)

Process:
  - Derive slug: lowercase opportunity.title, spaces → hyphens, strip punctuation
  - Fill C6 template fields
  - Express success criteria as testable outcomes (measurable or binary)
  - Resolve suggested-blueprint: must match a filename in plugins/sdlc/blueprints/; if none matches exactly, use closest and flag in SDLC Handoff Notes

Output:
  - File written to: .sdlc/briefs/<slug>.md
  - All C6 required fields populated; status: pending

Constraints:
  - One file per invocation (F5.AC1, F5.AC4)
  - suggested-blueprint must match a known blueprint name (NF2)
  - success-criteria entries must be testable outcomes (F5.AC3)
  - Writes only to .sdlc/briefs/<slug>.md (NF3)
```

**Dependencies**
- Internal: C6 (template structure)
- Existing: `plugins/sdlc/blueprints/` — read to validate suggested-blueprint value (NF2)

---

### C6 — Product Brief Template

**File:** `plugins/pm/templates/product-brief.md`
**Satisfies:** F5, NF2
**Responsibility:** Defines the canonical structure of a Product Brief whose fields map one-to-one to SDLC Phase 1 spec inputs, enabling SDLC to bootstrap a spec from a brief with zero clarifying questions.

**Interfaces**

```
Template structure (filled by C5):

---
status: pending
opportunity-title: <string>
category: <gap | improvement | moonshot>
impact: <high | medium | low>
effort: <high | medium | low>
suggested-blueprint: <name matching plugins/sdlc/blueprints/ filename>
source: <synthesized | human-sourced>
---

# Product Brief: <opportunity-title>

## Opportunity Statement
<One paragraph: the opportunity, its origin, why it matters.>

## Evidence
<Bullet list of supporting findings; each cites source (audit / competitive scan / human idea).>

## Success Criteria
<Numbered list of testable outcomes — measurable (with threshold) or binary (done/not done).>

## Scope
### In Scope
<Bullet list of what this brief covers.>
### Out of Scope
<Bullet list of explicit exclusions. Must contain at least one entry.>

## PRODUCT.md Goals Served
<List of PRODUCT.md goals this opportunity addresses. Required for moonshots.>

## SDLC Handoff Notes
<Optional: architectural hints, known constraints, or risks for the SDLC Phase 1 agent.>

Required sections: all except SDLC Handoff Notes (NF2)
Out of Scope must have ≥ 1 entry (mirrors SDLC spec requirement)
```

**Dependencies**
- Internal: none (consumed by C5)
- Existing: none

---

### C7 — Plugin Manifest

**File:** `plugins/pm/.claude-plugin/plugin.json`
**Satisfies:** F1, F2, F3, F4, F5, F6, F7
**Responsibility:** Declares the pm plugin's identity so the Claude Code plugin loader can register the `/pm` skill from the `plugins/pm/` directory.

**Interfaces**

```
Static JSON:
{
  "name": "pm",
  "version": "0.1.0",
  "description": "Product management skill: audits the codebase against product goals, scans the competitive landscape, synthesizes ranked opportunities, and produces Product Briefs that feed into the /sdlc skill.",
  "author": { "name": "<author>", "email": "<email>" }
}

Registration: the plugin entry in .claude-plugin/marketplace.json must reference this manifest
  via source.path: "plugins/pm"
Skill discovery: loader finds plugins/pm/skills/pm/SKILL.md by convention (skills/<name>/SKILL.md)
```

**Dependencies**
- Internal: none
- Existing: `.claude-plugin/marketplace.json` — pm plugin entry must be added (see Integration Points)

---

## Integration Points

| Existing File | Change | Rationale |
|---------------|--------|-----------|
| `.claude-plugin/marketplace.json` | Add pm plugin entry to plugins array with `source.path: "plugins/pm"` | Registers the pm plugin with the Claude Code loader so `/pm` can be invoked (F1, F2, F3, F4, F5, F6, F7, NF1, NF2, NF3) |
| `plugins/sdlc/blueprints/` | C5 reads this directory at runtime to validate `suggested-blueprint` values match known filenames | NF2 — briefs must reference a valid SDLC blueprint name |

---

## Constraints on Implementation

- **No writes outside the four permitted paths.** The PM skill may only write to: `PRODUCT.md` (bootstrap only, F1), `.sdlc/briefs/<slug>.md` (F5), `.sdlc/pm-log.md` (F6), and `.sdlc/pm-ideas.md` (checkbox updates only, F7). C2, C3, and C4 must not write files. Only C1 and C5 write files, to permitted paths only.
- **No web search outside C3.** C2, C4, and C5 must not invoke WebSearch or WebFetch. Only C3 may use web search, capped at 10 combined calls per run (NF1).
- **Agents are prompt files, not code.** All components (C1–C7) are markdown or JSON files. No Python, TypeScript, or shell scripts are introduced. All components follow the pattern in `plugins/sdlc/`.
- **PRODUCT.md bootstrap is one-shot.** C2 bootstrap mode generates a starter file and pauses for user review. After review, the pipeline continues. The PM skill does not offer to edit PRODUCT.md on subsequent runs.
- **Moonshot gate is a soft filter.** C4 must separate moonshots that cannot cite at least one PRODUCT.md goal into a `discarded-moonshots` section — they are shown to the user with a reason but excluded from the ranked list. This is mandatory separation, not silent discard.
- **suggested-blueprint must be a known name.** C5 must validate the suggested-blueprint value against filenames in `plugins/sdlc/blueprints/`. If no exact match, use closest and flag in SDLC Handoff Notes.
- **Plugin structure follows the SDLC plugin convention.** Layout: `plugins/pm/.claude-plugin/plugin.json`, `plugins/pm/skills/pm/SKILL.md`, `plugins/pm/agents/*.md`, `plugins/pm/templates/*.md`.

---

## Rejected Approaches

- **No bootstrap (noted, not a design choice):** No `claude-code-plugin` blueprint exists. Architecture was derived from the SDLC plugin as the closest reference. A `claude-code-plugin` blueprint should be created after this feature ships to avoid repeating this survey for future plugins.

- **Single monolithic SKILL.md with all logic inlined:** The full pipeline could be one large prompt. Rejected because: (a) combined context would exceed practical window sizes for non-trivial codebases; (b) the SDLC plugin demonstrates sub-agents provide better isolation; (c) separate agent files allow phase-by-phase testing without running the full pipeline.

- **Persisting competitive landscape as a separate file:** The plan mentions a `competitive-landscape.md` template persisted between runs. Rejected because the spec (F6) captures competitive scan summaries in `pm-log.md`; a separate file would require an additional write path not covered by NF3.

- **Parallel agent invocation (audit and scan simultaneously):** Running C2 and C3 in parallel would reduce wall-clock time. Rejected because C3 requires C2's feature-inventory as input (F3.AC1) — the scan must be scoped to the current product's feature set to identify gaps meaningfully.

- **Automated brief approval for low-risk types:** C4 could auto-approve improvement opportunities. Rejected because the spec explicitly marks automated brief approval as out of scope. C5 always writes `status: pending`.

---

## Gate: Architecture → Contract

**Validator checks (all must pass):**
- [ ] Every requirement ID from spec's frontmatter appears in this doc's `satisfies`
- [ ] Every requirement ID appears in the Coverage table
- [ ] Every component has: file path, satisfies list, responsibility, interfaces, dependencies
- [ ] Every integration point has a rationale referencing a requirement ID
- [ ] Constraints on Implementation is non-empty
- [ ] Rejected Approaches documents at least one alternative considered
