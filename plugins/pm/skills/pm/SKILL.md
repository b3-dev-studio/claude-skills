---
name: pm
description: Audits the codebase against product goals, scans the competitive landscape, synthesizes ranked opportunities, and produces Product Briefs that feed into the /sdlc skill.
argument-hint: "[focus=<area>]"
allowed-tools: [Read, Write, Edit, Glob, Grep, TodoWrite, Agent]
---

# PM Skill

Drive a complete product management cycle: audit the current product state, scan the competitive landscape, synthesize ranked opportunities, and produce Product Briefs ready for the `/sdlc` skill.

The output of this skill is one or more files in `.sdlc/briefs/` with `status: pending`. A human reviews and approves briefs. The `/sdlc` skill consumes approved briefs.

---

## Standing Context

Load these once at startup. Hold in working memory for the entire run.

| Source | What it provides |
|--------|-----------------|
| `PRODUCT.md` (project root) | Product vision, goals, success metrics, constraints |
| `.sdlc/pm-ideas.md` (if exists) | Human-provided ideas for evaluation |
| `.sdlc/decision-log.md` (if exists) | Architectural decisions — passed to the product auditor |

---

## Startup Checks

Run both checks before invoking any agent.

### Check 1: PRODUCT.md

Read `PRODUCT.md` from the project root.

**If missing:** invoke the `product-auditor` agent in **bootstrap mode** with the project root path. Present the generated draft to the user and pause for review. Once the user confirms the draft, write it to `PRODUCT.md`. Then continue with Check 2.

**If present:** load into working memory as `product-context`. Continue with Check 2.

### Check 2: pm-ideas.md

Read `.sdlc/pm-ideas.md` if it exists.

- Extract all unchecked items — lines matching the pattern `- [ ] ...`
- Store the extracted text of each unchecked item as the `human-ideas` list
- If the file does not exist, or contains no unchecked items, set `human-ideas` to an empty list

Do not pass checked items (`- [x] ...`) to the synthesizer — they have already been evaluated.

---

## Pipeline

Run the four agents sequentially. Each step depends on the output of the previous one.

### Step 1 — Product Audit

Invoke the `product-auditor` agent in **audit mode** with:
- `product-context`: the PRODUCT.md content from working memory
- Project root path
- `decision-log`: content of `.sdlc/decision-log.md` (empty string if absent)

Receive: `audit-report` (structured markdown). Hold in working memory.

### Step 2 — Competitive Scan

Invoke the `competitive-scanner` agent with:
- `product-context`: PRODUCT.md content
- `audit-report`: the feature-inventory section from Step 1
- `focus`: the value of the `focus=<area>` argument if provided; otherwise omit

Receive: `competitive-landscape` (structured markdown). Hold in working memory.

### Step 3 — Opportunity Synthesis

Invoke the `opportunity-synthesizer` agent with:
- `product-context`: PRODUCT.md content
- `audit-report`: full Step 1 output
- `competitive-landscape`: full Step 2 output
- `human-ideas`: the list from Startup Check 2 (may be empty)
- `focus`: forwarded from arguments if provided

Receive:
- `ranked-opportunities`: ordered list of opportunities
- `discarded-moonshots`: moonshots that failed the goal-citation gate (may be empty)
- `evaluated-ideas`: list of human idea strings that were processed

### Step 4 — User Selection

Present the `ranked-opportunities` to the user. If `discarded-moonshots` is non-empty, show it as a separate section after the ranked list.

Ask the user: **"Which opportunities would you like to generate Product Briefs for? Enter the rank numbers (e.g., 1, 3) or 'none' to skip brief generation."**

Wait for the user's response before proceeding.

### Step 5 — Brief Generation

For each opportunity selected by the user, invoke the `brief-writer` agent once with:
- `opportunity`: the selected ranked-opportunity entry
- `product-context`: PRODUCT.md content
- `audit-report`: full Step 1 output
- `competitive-landscape`: full Step 2 output

Each invocation writes one file to `.sdlc/briefs/<slug>.md`.

If the user selected 'none', skip this step.

---

## Post-Pipeline Writes

After the pipeline completes (whether or not briefs were generated):

### Write 1 — PM Log

Append a cycle summary to `.sdlc/pm-log.md`. Create the file if it does not exist. Do not modify any existing content — append only.

Cycle summary format:
```markdown
---
## PM Cycle — <ISO 8601 date>

**Product Domain:** <domain from PRODUCT.md>
**Focus:** <focus area if provided, otherwise "whole product">
**Opportunities Found:** <count from ranked-opportunities>
**Briefs Generated:** <count of briefs written in Step 5>

**Findings Summary:**
<One paragraph: what the audit found, what the competitive scan revealed, what the top opportunities are, and any notable discarded moonshots.>
```

### Write 2 — Idea Checkboxes

In `.sdlc/pm-ideas.md`, mark each item from the `evaluated-ideas` list (received in Step 3) as checked. Change `- [ ] <idea text>` to `- [x] <idea text>` for each matching line. Do not modify any other lines.

If `.sdlc/pm-ideas.md` does not exist or `evaluated-ideas` is empty, skip this write.

---

## Permitted Write Targets

This skill may only write to the following paths. All other files are **read-only**.

| Path | Condition |
|------|-----------|
| `PRODUCT.md` | Bootstrap mode only (Startup Check 1, when file was absent) |
| `.sdlc/briefs/<slug>.md` | One file per selected opportunity (Step 5, written by brief-writer) |
| `.sdlc/pm-log.md` | Append cycle summary after pipeline completes |
| `.sdlc/pm-ideas.md` | Checkbox updates only — change `- [ ]` to `- [x]` for evaluated ideas |

Writing to `.sdlc/current/`, `.sdlc/decision-log.md`, any source file, or any path not listed above is forbidden.
