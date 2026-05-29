---
artifact: contract
version: 1
gate_status: passed
satisfies: [C1, C2, C3, C4, C5, C6, C7, INT1]
---

# Implementation Contract: PM Skill (Product Management Plugin)

## Component Coverage

| Component | File | Action |
|-----------|------|--------|
| C1 | `plugins/pm/skills/pm/SKILL.md` | create |
| C2 | `plugins/pm/agents/product-auditor.md` | create |
| C3 | `plugins/pm/agents/competitive-scanner.md` | create |
| C4 | `plugins/pm/agents/opportunity-synthesizer.md` | create |
| C5 | `plugins/pm/agents/brief-writer.md` | create |
| C6 | `plugins/pm/templates/product-brief.md` | create |
| C7 | `plugins/pm/.claude-plugin/plugin.json` | create |
| INT1 | `.claude-plugin/marketplace.json` | modify |

---

## File Contracts

### `plugins/pm/skills/pm/SKILL.md` — create
**Implements:** C1
**Satisfies:** F1, F6, F7, NF3

**Required frontmatter fields — must be present with these exact keys and value constraints:**
```yaml
name: pm
description: <string — one sentence describing the PM skill's pipeline purpose>
argument-hint: "[focus=<area>]"
allowed-tools: [Read, Write, Edit, Glob, Grep, TodoWrite, Agent]
```

**Required sections — the file body must contain all of the following headings in order:**

1. `# PM Skill` — top-level heading introducing the skill
2. `## Standing Context` — lists the files to load at startup: `PRODUCT.md`, `.sdlc/pm-ideas.md`, `.sdlc/decision-log.md`
3. `## Startup Checks` — documents the PRODUCT.md bootstrap gate and pm-ideas.md parsing
4. `## Pipeline` — documents all five pipeline steps, which agent is invoked at each step, and what is passed/received
5. `## Post-Pipeline Writes` — documents the two post-pipeline writes: append to `.sdlc/pm-log.md` and mark evaluated ideas checked in `.sdlc/pm-ideas.md`
6. `## Permitted Write Targets` — explicitly lists the four permitted write paths and states that all other files are read-only

**Required behavioral constraints that must be stated in the prompt text:**

- Startup check 1: Read `PRODUCT.md` from the project root. If missing: invoke C2 in bootstrap mode, present output to user, and pause for review before continuing. If present: load into working memory as product-context.
- Startup check 2: Read `.sdlc/pm-ideas.md` if it exists. Extract all unchecked items (lines matching `- [ ] ...`). Pass as human-ideas list to C4. If the file is absent or contains no unchecked items, pass an empty list.
- Pipeline step order must be: C2 → C3 → C4 → user selection → C5 (per selected opportunity).
- After C4 output is produced, present ranked-opportunities to the user and receive user-selected opportunities before invoking C5.
- C5 is invoked once per selected opportunity; each invocation produces one brief file.
- After pipeline completes, append cycle summary to `.sdlc/pm-log.md` (create file if absent).
- After pipeline completes, mark each evaluated idea as `- [x]` in `.sdlc/pm-ideas.md`.
- The `focus` argument (if present) must be forwarded to C3 and C4; if absent, whole-product scope is used.
- The skill must not write to any file outside the four permitted paths.

---

### `plugins/pm/agents/product-auditor.md` — create
**Implements:** C2
**Satisfies:** F1, F2

**Required frontmatter fields:**
```yaml
name: product-auditor
description: <string — one sentence describing local-only codebase analysis and PRODUCT.md bootstrap>
tools: Read, Grep, Glob
model: sonnet
```

Note: `tools` must NOT include `WebSearch` or `WebFetch`.

**Required sections — the file body must contain all of the following headings:**

1. `# Product Auditor` — top-level heading
2. `## Invocation Modes` — introduces the two modes and when each is triggered
3. `## Mode A — Bootstrap` — specifies inputs, process, and required output structure when PRODUCT.md is absent
4. `## Mode B — Audit` — specifies inputs, process, and required output structure for normal runs
5. `## Output Format` — defines the structured markdown fields for each mode's output
6. `## Constraints` — states no-WebSearch and no-file-writes rules

**Required behavioral constraints that must be stated in the prompt text:**

- Mode A (Bootstrap): input is the project root directory path. Output must be a structured PRODUCT.md draft containing exactly these six sections: `## Vision`, `## Success Metrics`, `## Target Users`, `## Strategic Priorities`, `## Known Constraints`, `## Non-Goals`. No web search is used.
- Mode B (Audit): inputs are product-context (PRODUCT.md content), project root directory path, and decision-log content if present. Output is a structured audit-report with these named fields: `feature-inventory` (list of implemented capabilities), `goal-status` (table mapping each PRODUCT.md goal to `implemented`, `partial`, or `missing`), `strengths`, `gaps`, `decision-log-notes` (omitted if no decision log was provided).
- No `WebSearch` or `WebFetch` calls under any circumstances (F2.AC4).
- No file writes under any circumstances (NF3).

---

### `plugins/pm/agents/competitive-scanner.md` — create
**Implements:** C3
**Satisfies:** F3, NF1

**Required frontmatter fields:**
```yaml
name: competitive-scanner
description: <string — one sentence describing web-based competitive research capped at 10 search calls>
tools: Read, WebSearch, WebFetch
model: sonnet
```

**Required sections — the file body must contain all of the following headings:**

1. `# Competitive Scanner` — top-level heading
2. `## Inputs` — documents the three inputs: product-context, audit-report (feature-inventory section), and optional focus string
3. `## Process` — describes how comparables are identified and how the search budget is managed
4. `## Output Format` — defines the structured competitive-landscape fields
5. `## Search Budget` — states the hard cap of 10 combined WebSearch + WebFetch calls and instructs the agent to track and report `search-budget-used`
6. `## Constraints` — states no-file-writes rule

**Required behavioral constraints that must be stated in the prompt text:**

- WebSearch and WebFetch combined must not exceed 10 calls per invocation (NF1). The agent must count calls and stop web activity when the budget is exhausted.
- Must identify 3–5 comparable products (F3.AC1). Fewer than 3 is an error; more than 5 is not permitted.
- For each comparable, document at least one capability gap (what they have that the current product lacks) (F3.AC2).
- Identify at least one current product differentiator (F3.AC3).
- Surface at least one trend in the space.
- If `focus` is provided, scope the entire scan to the specified feature area rather than the whole product (F3.AC4).
- Output must include the field `search-budget-used: <integer>` showing how many WebSearch + WebFetch calls were made.
- Output format for competitive-landscape:
  - `comparables`: list of 3–5 entries, each with `name`, `url`, `capability-gaps` (list)
  - `differentiators`: list
  - `trends`: list
  - `search-budget-used`: integer (≤ 10)
- No file writes under any circumstances (NF3).

---

### `plugins/pm/agents/opportunity-synthesizer.md` — create
**Implements:** C4
**Satisfies:** F4, F7

**Required frontmatter fields:**
```yaml
name: opportunity-synthesizer
description: <string — one sentence describing cross-referencing audit and competitive results to produce a ranked opportunity list>
tools: Read
model: sonnet
```

Note: `tools` must NOT include `WebSearch` or `WebFetch`. `Write` and `Edit` must not appear.

**Required sections — the file body must contain all of the following headings:**

1. `# Opportunity Synthesizer` — top-level heading
2. `## Inputs` — documents all five inputs: product-context, audit-report, competitive-landscape, human-ideas list, optional focus
3. `## Derivation Rules` — describes how gap, improvement, and moonshot opportunities are derived
4. `## Moonshot Gate` — explicitly documents the soft filter: moonshots citing ≥ 1 PRODUCT.md goal enter ranked-opportunities; moonshots with no valid citation go to discarded-moonshots with reason stated
5. `## Human Idea Handling` — describes how human-ideas are classified, filtered through the moonshot gate (if applicable), scored, and ranked alongside synthesized opportunities; each is flagged as `human-sourced`
6. `## Ranking` — describes impact-to-effort ratio scoring (highest first)
7. `## Output Format` — defines all three output structures
8. `## Constraints` — states no-web-search and no-file-writes rules

**Required behavioral constraints that must be stated in the prompt text:**

- Derive gap opportunities from audit gaps and competitive capability-gaps.
- Derive improvement opportunities from feature-inventory and competitive comparisons.
- Generate moonshot opportunities from competitive trends; apply the moonshot gate.
- Moonshot gate is mandatory (not optional): moonshots that cite at least one PRODUCT.md goal are retained in ranked-opportunities; moonshots with no valid goal citation are placed in discarded-moonshots with the reason (e.g., "no PRODUCT.md goal citation found") (F4.AC3, F4.AC5).
- Evaluate every human-idea: classify as gap, improvement, or moonshot; apply moonshot gate if classified as moonshot; score and rank alongside synthesized opportunities; flag each as `source: human-sourced` (F7.AC1).
- Rank all retained opportunities by impact-to-effort ratio, highest first (F4.AC4).
- Return the `evaluated-ideas` list (the human-ideas that were evaluated this run) to C1 for checkbox marking (F7.AC3).
- If `focus` is provided, scope synthesis to the specified feature area.
- Output structures:
  - `ranked-opportunities`: list, each entry with fields: `rank`, `title`, `category` (gap|improvement|moonshot), `source` (synthesized|human-sourced), `impact` (high|medium|low), `effort` (high|medium|low), `summary`, `goal-citations`, `evidence`
  - `discarded-moonshots`: list, each entry with fields: `title`, `source` (synthesized|human-sourced), `reason`
  - `evaluated-ideas`: list of idea titles/text that were processed this run
- No WebSearch or WebFetch calls (NF3).
- No file writes (NF3).

---

### `plugins/pm/agents/brief-writer.md` — create
**Implements:** C5
**Satisfies:** F5, NF2

**Required frontmatter fields:**
```yaml
name: brief-writer
description: <string — one sentence describing filling the product-brief template and writing to .sdlc/briefs/>
tools: Read, Write, Glob
model: sonnet
```

**Required sections — the file body must contain all of the following headings:**

1. `# Brief Writer` — top-level heading
2. `## Inputs` — documents all four inputs: opportunity (single ranked-opportunity entry), product-context, audit-report, competitive-landscape
3. `## Slug Derivation` — states the rule: lowercase opportunity title, spaces to hyphens, strip punctuation
4. `## Blueprint Validation` — instructs the agent to read `plugins/sdlc/blueprints/` to list known filenames and select an exact match; if no exact match, use closest and flag in SDLC Handoff Notes
5. `## Template Filling` — describes how each C6 section is populated
6. `## Output` — states that the output is a file written to `.sdlc/briefs/<slug>.md` with `status: pending`
7. `## Constraints` — states one-file-per-invocation, testable success criteria, permitted write path

**Required behavioral constraints that must be stated in the prompt text:**

- This agent is invoked once per selected opportunity; it produces exactly one file per invocation (F5.AC1, F5.AC4).
- The output file path is `.sdlc/briefs/<slug>.md` where slug is derived from the opportunity title (lowercase, spaces to hyphens, punctuation stripped) (F5.AC1).
- The file must be written with frontmatter field `status: pending`. The agent must not write `status: approved` (spec Out of Scope).
- All required C6 template fields must be populated; no field may be left blank or contain a placeholder.
- `suggested-blueprint` must be a filename found in `plugins/sdlc/blueprints/` (without the `.md` extension). Known valid values at time of contract: `rest-api-resource`, `background-worker`, `event-driven-handler`, `webhook-receiver`, `oauth-integration`. If no exact match exists, use the closest and add a note in `## SDLC Handoff Notes` (NF2).
- Each success criterion must be expressed as a testable outcome — either measurable (with a threshold or metric) or binary (done/not done) (F5.AC3).
- The agent writes only to `.sdlc/briefs/<slug>.md`. It must not write to any other path (NF3).

---

### `plugins/pm/templates/product-brief.md` — create
**Implements:** C6
**Satisfies:** F5, NF2

This is a template file consumed by C5. It defines the canonical structure that every generated brief must follow. The file contains the template scaffold with placeholder text; C5 fills it at runtime.

**Required frontmatter fields (template form — these appear in every generated brief):**
```yaml
status: pending
opportunity-title: <string>
category: <gap | improvement | moonshot>
impact: <high | medium | low>
effort: <high | medium | low>
suggested-blueprint: <name matching plugins/sdlc/blueprints/ filename without .md extension>
source: <synthesized | human-sourced>
```

**Required sections — the template body must contain all of the following headings:**

1. `# Product Brief: <opportunity-title>` — top-level heading with placeholder for title
2. `## Opportunity Statement` — placeholder: one paragraph describing the opportunity, its origin, and why it matters
3. `## Evidence` — placeholder: bullet list of supporting findings, each citing its source (audit / competitive scan / human idea)
4. `## Success Criteria` — placeholder: numbered list of testable outcomes, each measurable or binary
5. `## Scope` — parent heading containing two required sub-headings:
   - `### In Scope` — placeholder: bullet list of what this brief covers
   - `### Out of Scope` — placeholder: bullet list of explicit exclusions; must contain at least one entry when filled
6. `## PRODUCT.md Goals Served` — placeholder: list of PRODUCT.md goals this opportunity addresses; required for moonshot category
7. `## SDLC Handoff Notes` — placeholder: optional architectural hints, known constraints, or risks for the SDLC Phase 1 agent

**Required structural constraints stated in the template:**

- All sections except `## SDLC Handoff Notes` are required; C5 must populate them (NF2).
- `### Out of Scope` must contain at least one entry when filled by C5.
- Placeholder text must make the requirement for each field unambiguous (e.g., note that success criteria must be measurable or binary).

---

### `plugins/pm/.claude-plugin/plugin.json` — create
**Implements:** C7
**Satisfies:** F1, F2, F3, F4, F5, F6, F7

**Required content — exact JSON structure:**
```json
{
  "name": "pm",
  "version": "0.1.0",
  "description": "Product management skill: audits the codebase against product goals, scans the competitive landscape, synthesizes ranked opportunities, and produces Product Briefs that feed into the /sdlc skill.",
  "author": {
    "name": "Brian Connelly",
    "email": "bjbb.connelly@gmail.com"
  }
}
```

The `name` field must be exactly `"pm"`. The `version` must be exactly `"0.1.0"`. The `description` must be exactly as shown above. The `author` object must contain `name` and `email`.

No additional fields are permitted. This is a static JSON file; no template placeholders remain.

---

### `.claude-plugin/marketplace.json` — modify
**Implements:** INT1
**Satisfies:** F1, F2, F3, F4, F5, F6, F7, NF1, NF2, NF3
**Change:** Add a second entry to the `plugins` array for the pm plugin.

**Unchanged content** *(must not be altered):*
- `$schema`
- `name` (marketplace name: `"brian-local-plugins"`)
- `description`
- `owner`
- The existing `sdlc` plugin entry (all fields)

**New entry to add to the `plugins` array:**
```json
{
  "name": "pm",
  "description": "Product management skill: audits the codebase against product goals, scans the competitive landscape, synthesizes ranked opportunities, and produces Product Briefs that feed into the /sdlc skill.",
  "author": {
    "name": "Brian Connelly"
  },
  "category": "development",
  "source": {
    "source": "git-subdir",
    "url": "file:///Users/brian/dev/claude-skills",
    "path": "plugins/pm",
    "ref": "master"
  }
}
```

The `source.path` must be `"plugins/pm"`. This enables the Claude Code plugin loader to find the manifest at `plugins/pm/.claude-plugin/plugin.json` and discover the skill at `plugins/pm/skills/pm/SKILL.md` by the `skills/<name>/SKILL.md` convention.

---

## Test Plan

All tests are manual verification steps. Run each test in a clean checkout with a real or representative codebase. "Verify" means observe the stated outcome directly.

| Test ID | Covers | File | Description |
|---------|--------|------|-------------|
| T1 | F1.AC1 | `plugins/pm/skills/pm/SKILL.md` | Run `/pm` in a project with no `PRODUCT.md`. Verify a starter `PRODUCT.md` is written to the project root and its contents are shown to the user before any audit or competitive scan begins. |
| T2 | F1.AC2 | `plugins/pm/skills/pm/SKILL.md` | Run `/pm` in a project where `PRODUCT.md` already exists. Verify the file is read, no bootstrap file is generated, and the pipeline proceeds directly to the audit step. |
| T3 | F1.AC3 | `plugins/pm/agents/product-auditor.md` | Trigger bootstrap mode (PRODUCT.md absent). Inspect the generated `PRODUCT.md`. Verify it contains all six sections: `## Vision`, `## Success Metrics`, `## Target Users`, `## Strategic Priorities`, `## Known Constraints`, `## Non-Goals`. |
| T4 | F2.AC1 | `plugins/pm/agents/product-auditor.md` | Run `/pm` against a codebase with multiple implemented features. Verify the audit-report's `feature-inventory` field is a list of implemented capabilities derived from the codebase (not from PRODUCT.md goals). |
| T5 | F2.AC2 | `plugins/pm/agents/product-auditor.md` | Run `/pm` with a `PRODUCT.md` containing at least three goals. Verify the audit-report's `goal-status` table maps each goal to one of: `implemented`, `partial`, or `missing`. |
| T6 | F2.AC3 | `plugins/pm/agents/product-auditor.md` | Run `/pm` in a project where `.sdlc/decision-log.md` exists with at least one entry. Verify the audit-report includes a `decision-log-notes` section referencing content from the decision log. |
| T7 | F2.AC4 | `plugins/pm/agents/product-auditor.md` | Run `/pm` and observe the audit step. Verify no WebSearch or WebFetch calls appear in the tool-use trace during C2 invocation. |
| T8 | F3.AC1 | `plugins/pm/agents/competitive-scanner.md` | Run `/pm` and observe C3 output. Verify the `comparables` list contains between 3 and 5 entries, each with a `name` and `url`. |
| T9 | F3.AC2 | `plugins/pm/agents/competitive-scanner.md` | Inspect the competitive-landscape output. Verify each comparable entry includes at least one `capability-gaps` item describing something they have that the current product lacks. |
| T10 | F3.AC3 | `plugins/pm/agents/competitive-scanner.md` | Inspect the competitive-landscape output. Verify the `differentiators` list contains at least one entry. |
| T11 | F3.AC4 | `plugins/pm/skills/pm/SKILL.md` | Run `/pm focus=authentication`. Verify the competitive-landscape output describes comparables and gaps scoped to the authentication feature area, and that the scan does not broadly cover unrelated product areas. |
| T12 | F4.AC1 | `plugins/pm/agents/opportunity-synthesizer.md` | Run `/pm` and inspect the ranked-opportunities output. Verify that opportunities are present from at least two of the three categories: `gap`, `improvement`, `moonshot`. |
| T13 | F4.AC2 | `plugins/pm/agents/opportunity-synthesizer.md` | Inspect each ranked opportunity. Verify every entry has both an `impact` field (high/medium/low) and an `effort` field (high/medium/low). |
| T14 | F4.AC3 | `plugins/pm/agents/opportunity-synthesizer.md` | Run `/pm` with a PRODUCT.md that has defined goals. Introduce a scenario where at least one moonshot idea does not map to any stated goal. Verify that moonshot appears in `discarded-moonshots` and not in `ranked-opportunities`. |
| T15 | F4.AC4 | `plugins/pm/agents/opportunity-synthesizer.md` | Inspect the ranked-opportunities list. Verify opportunities are ordered so that higher impact-to-effort ratio entries appear before lower ones (e.g., a high-impact/low-effort entry ranks above a low-impact/high-effort entry). |
| T16 | F4.AC5 | `plugins/pm/agents/opportunity-synthesizer.md` | Inspect the discarded-moonshots section when it contains at least one entry. Verify each entry shows: the idea title, its `source` (synthesized or human-sourced), and a `reason` string explaining why it was discarded. |
| T17 | F5.AC1 | `plugins/pm/agents/brief-writer.md` | Select one opportunity for brief generation. Verify a file is created at `.sdlc/briefs/<slug>.md` where the slug is derived from the opportunity title, and the file's frontmatter contains `status: pending`. |
| T18 | F5.AC2 | `plugins/pm/agents/brief-writer.md` | Open a generated brief. Verify it contains all required fields: `## Opportunity Statement`, `## Evidence`, `## Success Criteria`, `### In Scope`, `### Out of Scope`, and the `suggested-blueprint` frontmatter field. |
| T19 | F5.AC3 | `plugins/pm/agents/brief-writer.md` | Inspect the `## Success Criteria` section of a generated brief. Verify each criterion is either measurable (includes a threshold or metric) or binary (states a done/not-done condition). No vague or subjective criteria are present. |
| T20 | F5.AC4 | `plugins/pm/agents/brief-writer.md` | Select two different opportunities for brief generation in the same run. Verify two distinct files are written to `.sdlc/briefs/`, each with a unique slug derived from its respective opportunity title. |
| T21 | F6.AC1 | `plugins/pm/skills/pm/SKILL.md` | Complete a full PM run. Verify `.sdlc/pm-log.md` exists after the run and contains a new cycle summary entry. If the file did not exist before the run, verify it was created. |
| T22 | F6.AC2 | `plugins/pm/skills/pm/SKILL.md` | Run `/pm` twice against the same project. After the second run, open `.sdlc/pm-log.md`. Verify the first run's entry is unchanged and a second entry has been appended below it. |
| T23 | F6.AC3 | `plugins/pm/skills/pm/SKILL.md` | Inspect a cycle summary entry in `.sdlc/pm-log.md`. Verify it contains: an ISO-format date, the product domain, the count of opportunities found, the count of briefs generated, and a paragraph summarizing findings. |
| T24 | F7.AC1 | `plugins/pm/skills/pm/SKILL.md` | Create `.sdlc/pm-ideas.md` with at least two unchecked items (`- [ ] idea text`). Run `/pm`. Verify both ideas appear in the ranked-opportunities or discarded-moonshots output with `source: human-sourced`. |
| T25 | F7.AC2 | `plugins/pm/skills/pm/SKILL.md` | Create `.sdlc/pm-ideas.md` containing only checked items (`- [x] idea text`). Run `/pm`. Verify no human-sourced ideas appear in the synthesis output and the tool-use trace shows no ideas were passed to C4. |
| T26 | F7.AC3 | `plugins/pm/skills/pm/SKILL.md` | After a run where unchecked ideas were evaluated, open `.sdlc/pm-ideas.md`. Verify each idea that was evaluated now reads `- [x]` instead of `- [ ]`. |
| T27 | F7.AC4 | `plugins/pm/skills/pm/SKILL.md` | Run `/pm` in a project where `.sdlc/pm-ideas.md` does not exist. Verify the skill completes the full pipeline without error and without any human-sourced ideas in the synthesis output. |
| T28 | F7.AC5 | `plugins/pm/skills/pm/SKILL.md` | After a run that marked ideas as `- [x]`, manually edit `.sdlc/pm-ideas.md` to uncheck one item (change `- [x]` back to `- [ ]`). Run `/pm` again. Verify that unchecked idea is re-evaluated and appears in the synthesis output as `human-sourced`. |

*Every acceptance criteria ID from the spec appears in the Covers column above: F1.AC1–F1.AC3, F2.AC1–F2.AC4, F3.AC1–F3.AC4, F4.AC1–F4.AC5, F5.AC1–F5.AC4, F6.AC1–F6.AC3, F7.AC1–F7.AC5.*

---

## Forbidden

- **No code files.** No Python, TypeScript, JavaScript, shell scripts, or any executable files may be created. All components are markdown (`.md`) or JSON (`.json`) files only.
- **No WebSearch or WebFetch in C2 (product-auditor.md).** The product auditor must perform local-only analysis. These tools must not appear in its `tools` frontmatter field and must not be invoked in its prompt logic.
- **No WebSearch or WebFetch in C4 (opportunity-synthesizer.md).** The synthesizer derives opportunities from already-collected inputs. These tools must not appear in its `tools` frontmatter field.
- **No WebSearch or WebFetch in C5 (brief-writer.md).** The brief writer fills a template from existing inputs. These tools must not appear in its `tools` frontmatter field.
- **No Write or Edit tools in C2 (product-auditor.md).** This agent produces its audit-report as output to C1; it must not write files directly.
- **No Write or Edit tools in C3 (competitive-scanner.md).** This agent produces competitive-landscape as output to C1; it must not write files directly.
- **No Write or Edit tools in C4 (opportunity-synthesizer.md).** This agent produces ranked-opportunities as output to C1; it must not write files directly.
- **No writes outside the four permitted paths.** The only permitted write targets across all components are: `PRODUCT.md` (C1, bootstrap mode only), `.sdlc/briefs/<slug>.md` (C5 only), `.sdlc/pm-log.md` (C1, append only), `.sdlc/pm-ideas.md` (C1, checkbox updates only). No other file may be written or edited by any component.
- **No writes to `.sdlc/current/`, `.sdlc/decision-log.md`, or any source file.** These are explicitly excluded by NF3.
- **No `status: approved` in generated briefs.** C5 must always write `status: pending`. Automated brief approval is out of scope.
- **No suggested-blueprint value not present in `plugins/sdlc/blueprints/`.** The only valid values at contract time (without the `.md` extension) are: `rest-api-resource`, `background-worker`, `event-driven-handler`, `webhook-receiver`, `oauth-integration`. A value not on this list requires a flag in `## SDLC Handoff Notes` and use of the closest matching name.
- **No modification to the existing `sdlc` plugin entry in `.claude-plugin/marketplace.json`.** The INT1 change adds a new array entry only; all existing content is preserved verbatim.
- **No PRODUCT.md editing after bootstrap.** C1 must not offer to edit or rewrite `PRODUCT.md` during a run where it already existed. After bootstrap, `PRODUCT.md` is read-only to the skill.
- **No parallel agent invocation.** C3 depends on C2's feature-inventory output; the pipeline must be strictly sequential: C2 → C3 → C4 → C5.
- **No re-evaluation of checked ideas.** Ideas in `.sdlc/pm-ideas.md` marked `- [x]` must be skipped entirely; they must not be passed to C4 (F7.AC2).

---

## Build Sequence

Implement files in this order (dependency-first):

1. `plugins/pm/templates/product-brief.md` — C6 (template) — no internal dependencies; consumed by C5
2. `plugins/pm/.claude-plugin/plugin.json` — C7 (manifest) — no internal dependencies; static JSON
3. `plugins/pm/agents/product-auditor.md` — C2 (leaf agent) — no internal agent dependencies; reads project files via tool calls
4. `plugins/pm/agents/competitive-scanner.md` — C3 (leaf agent) — no internal agent dependencies; uses web tools and C2 output as input (data dependency, not file dependency)
5. `plugins/pm/agents/opportunity-synthesizer.md` — C4 (leaf agent) — no internal agent dependencies; uses C2 and C3 outputs as input (data dependency, not file dependency)
6. `plugins/pm/agents/brief-writer.md` — C5 (mid-tier agent) — depends on C6 template structure
7. `plugins/pm/skills/pm/SKILL.md` — C1 (orchestrator) — depends on C2, C3, C4, C5; must be implemented last among the pm files
8. `.claude-plugin/marketplace.json` — INT1 (integration) — depends on C7 being complete so the path `plugins/pm` is valid to reference

---

## Gate: Contract → Implementation

**Validator checks (all must pass):**
- [ ] Every component ID from arch's frontmatter appears in this doc's `satisfies`
- [ ] Every component appears in the Coverage table
- [ ] Every file contract has explicit required signatures (frontmatter fields and section headings)
- [ ] For modify actions: unchanged exports are listed
- [ ] Every acceptance criteria ID from spec has ≥1 row in the Test Plan
- [ ] Forbidden list is non-empty
