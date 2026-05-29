# Vocabulary

Canonical domain terms for this project. All spec, arch, and contract documents must use these terms.

---

## Product Brief
A structured markdown file in `.sdlc/briefs/` that describes a single product opportunity. Written by the PM skill's `brief-writer` agent. Consumed by the SDLC skill as Phase 1 input. Fields: opportunity statement, evidence, success criteria, scope, suggested blueprint, status. Status lifecycle: `pending` → `approved` → `in-progress` → `complete` | `deferred`.

## PRODUCT.md
The PM equivalent of `CLAUDE.md`. A human-authored file in the project root defining: product vision, success metrics, target users, strategic priorities, known constraints, and non-goals. Read at the start of every PM run. The anchor against which all analysis is measured.

## PM Cycle
One complete execution of the `/pm` skill: audit → competitive scan → synthesis → brief generation → log update. Results in zero or more Product Briefs written to `.sdlc/briefs/`.

## pm-log.md
An append-only file at `.sdlc/pm-log.md` recording the history of PM cycle runs. Each entry is a structured summary: date, domain, opportunities found, briefs generated, findings paragraph. Written by the PM skill orchestrator at the end of each cycle.

## pm-ideas.md
A human-maintained file at `.sdlc/pm-ideas.md` containing product ideas for the PM skill to evaluate. Uses markdown checkbox syntax: `- [ ]` for pending ideas, `- [x]` for already-evaluated ideas. The PM skill reads unchecked items, passes them to the opportunity-synthesizer, and marks them checked after evaluation.

## Product Auditor
The C2 agent in the PM skill. Performs local-only codebase analysis. Two modes: Bootstrap (generates a starter PRODUCT.md when none exists) and Audit (produces a structured audit-report comparing the current implementation against PRODUCT.md goals).

## Competitive Scanner
The C3 agent in the PM skill. Uses web search (capped at 10 calls) to identify 3–5 comparable products, document capability gaps, surface differentiators, and note trends.

## Opportunity Synthesizer
The C4 agent in the PM skill. Cross-references the audit report, competitive landscape, PRODUCT.md goals, and human ideas to produce a ranked list of gap, improvement, and moonshot opportunities. Applies the moonshot gate.

## Moonshot Gate
A soft filter applied by the Opportunity Synthesizer to moonshot opportunities. A moonshot that cites at least one PRODUCT.md goal passes and enters `ranked-opportunities`. A moonshot with no valid goal citation fails and appears in `discarded-moonshots` with a stated reason. No moonshot is silently discarded.

## Brief Writer
The C5 agent in the PM skill. Invoked once per selected opportunity. Fills the Product Brief template and writes one file to `.sdlc/briefs/<slug>.md` with `status: pending`.

## Audit Report
Structured markdown output from the Product Auditor (Mode B). Contains: feature-inventory, goal-status table, strengths, gaps, decision-log-notes. Held in C1 working memory and passed to C3 and C4.

## Competitive Landscape
Structured markdown output from the Competitive Scanner. Contains: comparables (3–5, each with capability gaps), differentiators, trends, search-budget-used. Held in C1 working memory and passed to C4.

## suggested-blueprint
A frontmatter field in every Product Brief. Names an SDLC blueprint (without `.md` extension) that the SDLC skill should use as its Phase 2 starting point. Must match a filename in `plugins/sdlc/blueprints/`. Valid values at current state: `rest-api-resource`, `background-worker`, `event-driven-handler`, `webhook-receiver`, `oauth-integration`.
