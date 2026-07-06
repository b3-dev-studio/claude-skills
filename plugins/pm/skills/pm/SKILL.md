---
name: pm
description: Audits the codebase against product goals, scans the competitive landscape, synthesizes ranked opportunities, and produces Product Briefs that feed into the /sdlc skill.
argument-hint: "[focus=<area>] [--auto [--interval=<duration>] [--top=<N>]]"
allowed-tools: [Read, Write, Edit, Glob, Grep, TodoWrite, Agent, Skill]
---

# PM Skill

Drive a complete product management cycle: audit the current product state, scan the competitive landscape, synthesize ranked opportunities, and produce Product Briefs ready for the `/sdlc` skill.

The output of this skill is one or more files in `.sdlc/briefs/` with `status: pending`. A human reviews and approves briefs. The `/sdlc` skill consumes approved briefs.

---

## Argument Parsing

Parse `$ARGUMENTS` before any other step.

0. If `--help` is present, print the following and stop:

   ```
   Usage: /pm [focus=<area>] [flags]

   Flags:
     --auto              Skip the opportunity selection prompt (Step 4) and generate
                         briefs for all ranked opportunities automatically.
     --interval=<dur>    Split the pipeline into two scheduled runs separated by <dur>
                         (e.g. 30m, 2h). Only applies when --auto is set.
                         Run 1: research pipeline (Steps 1–3).
                         Run 2: brief generation (Step 5).
     --top=<N>           In auto mode, generate briefs for the top N opportunities only.

   Arguments:
     focus=<area>        Narrow the audit and competitive scan to a specific product area.

   Modes:
     (no flags)          Adhoc — pauses at Step 4 for you to select which briefs to generate.
     --auto              Auto, single session — selects all (or top N) opportunities and runs
                         start to finish without prompting.
     --auto --interval   Auto, split — Run 1 does the research and schedules Run 2 for
                         brief generation after the delay. State is saved to .sdlc/pm-state.md.

   Examples:
     /pm
     /pm focus=auth
     /pm --auto
     /pm --auto --top=3
     /pm --auto --interval=1h
     /pm --auto --interval=2h --top=2 focus=onboarding
   ```

1. Detect flags:
   - `--auto` present → `mode = auto`
   - `--auto` absent → `mode = adhoc`
   - `--interval=<duration>` present → `schedule_interval = <duration>` (e.g., `30m`, `2h`)
   - `--interval` absent when `mode = auto` → `schedule_interval` is unset (single-session run)
   - `--top=<N>` present → `auto_top = N` (generate briefs for top N opportunities only)
   - `--top` absent → `auto_top` is unset (generate briefs for all opportunities)
   - `focus=<area>` → `focus = <area>` (forwarded to agents)
2. Strip all flags from `$ARGUMENTS`.

**Adhoc mode** (default): Step 4 pauses and asks the user which opportunities to generate briefs for. The full pipeline runs in a single session.

**Auto mode**: Step 4 is skipped — opportunities are auto-selected (all, or top N if `--top` is set). If `--interval` is also set, the pipeline splits into two scheduled runs: Run 1 covers Steps 1–3 (research) and Run 2 covers Step 5 (brief generation). State is persisted to `.sdlc/pm-state.md` between runs.

---

## Resume Detection

Before running Startup Checks, look for `.sdlc/pm-state.md`.

- **File exists with `status: post-synthesis`** → a prior auto-mode run completed Steps 1–3. Load `audit-report`, `competitive-landscape`, `ranked-opportunities`, `discarded-moonshots`, and `evaluated-ideas` from the file into working memory. Skip Startup Checks, Steps 1–3, and Step 4. Jump directly to **Step 5**.
- **File absent or `status` is not `post-synthesis`** → fresh run. Proceed to Startup Checks.

---

## Standing Context

Load these once at startup. Hold in working memory for the entire run.

| Source | What it provides |
|--------|-----------------|
| `PRODUCT.md` (project root) | Product vision, goals, success metrics, constraints |
| `.sdlc/pm-ideas.md` (if exists) | Human-provided ideas for evaluation |
| `.sdlc/decision-log.md` (if exists) | Architectural decisions — passed to the product auditor |
| `.sdlc/briefs/` filenames + each brief's `opportunity-title` and `status` frontmatter (if any exist) | Prior briefs — used by the synthesizer and Step 5 to avoid re-proposing opportunities already briefed or implemented |
| Last 2 cycle summaries in `.sdlc/pm-log.md` (if exists) | What recent cycles already found, so findings aren't recycled |

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
- `focus`: the value of `focus` if provided; otherwise omit

Receive: `competitive-landscape` (structured markdown). Hold in working memory.

### Step 3 — Opportunity Synthesis

Invoke the `opportunity-synthesizer` agent with:
- `product-context`: PRODUCT.md content
- `audit-report`: full Step 1 output
- `competitive-landscape`: full Step 2 output
- `human-ideas`: the list from Startup Check 2 (may be empty)
- `prior-briefs`: the list of existing brief titles and statuses from Standing Context (may be empty) — instruct the synthesizer to exclude opportunities substantially covered by an existing brief of any status
- `focus`: forwarded from arguments if provided

Receive:
- `ranked-opportunities`: ordered list of opportunities
- `discarded-moonshots`: moonshots that failed the goal-citation gate (may be empty)
- `evaluated-ideas`: list of human idea strings that were processed

**Continuation after Step 3:**
- **Adhoc mode:** Proceed to Step 4.
- **Auto mode, `--interval` set:** Write `.sdlc/pm-state.md` (see format below). Run Post-Pipeline Write 2 (idea checkboxes) now, since evaluated-ideas won't be available in Run 2. Invoke the `schedule` skill to run `/pm --auto --interval={{schedule_interval}}{{#auto_top}} --top={{auto_top}}{{/auto_top}}{{#focus}} focus={{focus}}{{/focus}}` in `{{schedule_interval}}`. Notify the user: "Steps 1–3 (research pipeline) complete. Brief generation scheduled in {{schedule_interval}}." Stop.
- **Auto mode, no `--interval`:** Proceed directly to Step 4 (auto-selection) without pausing.

#### pm-state.md format

```markdown
---
status: post-synthesis
date: <ISO 8601 date>
---

## audit-report
<full Step 1 output>

---

## competitive-landscape
<full Step 2 output>

---

## ranked-opportunities
<full ranked-opportunities list>

---

## discarded-moonshots
<full discarded-moonshots list, or "(none)">

---

## evaluated-ideas
<one idea per line>
```

### Step 4 — Opportunity Selection

**Adhoc mode:** Present the `ranked-opportunities` to the user. If `discarded-moonshots` is non-empty, show it as a separate section after the ranked list. Ask: **"Which opportunities would you like to generate Product Briefs for? Enter the rank numbers (e.g., 1, 3) or 'none' to skip brief generation."** Wait for the user's response before proceeding.

**Auto mode:** Auto-select opportunities without prompting the user.
- If `auto_top` is set: select the top `auto_top` entries from `ranked-opportunities`.
- If `auto_top` is unset: select all entries from `ranked-opportunities`.
- Notify the user which opportunities were selected.

### Step 5 — Brief Generation

**Dedup check first:** for each selected opportunity, compare against the `prior-briefs` list. If an existing brief in `.sdlc/briefs/` already covers substantially the same opportunity, skip it and note the skip (with the existing brief's filename) in the cycle summary instead of writing a duplicate.

For each remaining selected opportunity, invoke the `brief-writer` agent once with:
- `opportunity`: the selected ranked-opportunity entry
- `product-context`: PRODUCT.md content
- `audit-report`: full Step 1 output
- `competitive-landscape`: full Step 2 output

Each invocation writes one file to `.sdlc/briefs/<slug>.md`.

If no opportunities were selected (adhoc: user chose 'none'; auto: ranked list was empty), skip this step.

**After Step 5 in auto mode with `--interval`:** Delete `.sdlc/pm-state.md` — the run state is no longer needed.

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

Note: in auto mode with `--interval`, Write 2 runs at the end of Run 1 (after Step 3) so that idea checkboxes are updated even if Run 2 never executes.

---

## Permitted Write Targets

This skill may only write to the following paths. All other files are **read-only**.

| Path | Condition |
|------|-----------|
| `PRODUCT.md` | Bootstrap mode only (Startup Check 1, when file was absent) |
| `.sdlc/briefs/<slug>.md` | One file per selected opportunity (Step 5, written by brief-writer) |
| `.sdlc/pm-log.md` | Append cycle summary after pipeline completes |
| `.sdlc/pm-ideas.md` | Checkbox updates only — change `- [ ]` to `- [x]` for evaluated ideas |
| `.sdlc/pm-state.md` | Auto mode with `--interval` only — written after Step 3, deleted after Step 5 |

Writing to `.sdlc/current/`, `.sdlc/decision-log.md`, any source file, or any path not listed above is forbidden.
