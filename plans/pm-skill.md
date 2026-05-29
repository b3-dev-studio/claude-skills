# Plan: Product Management Skill

## Overview

A Claude Code plugin that evaluates the current product/software against its goals, the competitive landscape, and hypothetical moonshot ideas — then produces structured Product Briefs that feed directly into the `/sdlc` skill. The two skills form a closed loop of autonomous continuous improvement.

---

## Core Principle

The PM skill is to *what to build* as the SDLC skill is to *how to build it*. They are separate plugins connected by a single artifact interface: the Product Brief. PM produces briefs. SDLC consumes them.

---

## Architecture

### The Continuous Improvement Loop

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   /pm                                                   │
│   ├── Audit: codebase + decision log                    │
│   ├── Competitive scan: web research                    │
│   ├── Goal gap analysis: PRODUCT.md                     │
│   ├── Opportunity identification                        │
│   ├── Moonshot generation                               │
│   └── Produces: Product Brief(s) → .sdlc/briefs/        │
│                        │                                │
│            [human reviews + approves]                   │
│                        │                                │
│                        ▼                                │
│   /sdlc (reads approved brief as Phase 1 input)         │
│   └── Phases 1–7: spec → arch → contract → implement    │
│                        │                                │
│       Phase 7 updates: decision-log, vocabulary,        │
│       bootstraps, CHANGELOG                             │
│                        │                                │
│                        └──────────────────────────────▶ │
│                           (next /pm run reads updated   │
│                            state and re-audits)         │
└─────────────────────────────────────────────────────────┘
```

### Relationship to SDLC Skill

| Concern | PM Skill | SDLC Skill |
|---------|----------|------------|
| What to build | ✓ | |
| Why to build it | ✓ | |
| Competitive context | ✓ | |
| How to build it | | ✓ |
| Structural correctness | | ✓ |
| Verified implementation | | ✓ |

**Interface:** `.sdlc/briefs/` directory. PM writes briefs here. Human approves. SDLC reads the approved brief as its Phase 1 input instead of free-text arguments.

---

## Persistent State

### `PRODUCT.md` (human-authored, like `CLAUDE.md`)
The PM equivalent of `CLAUDE.md`. Defines:
- Product vision and north star
- Success metrics (quantitative where possible)
- Target users and their core problems
- Strategic priorities for the current period
- Known constraints (technical, business, resource)
- Explicit non-goals

This file is read at the start of every PM run. It is the anchor against which all analysis is measured.

### `.sdlc/briefs/` (PM-written, human-approved)
Queue of Product Briefs. Status tracked in frontmatter: `pending`, `approved`, `in-progress`, `complete`, `deferred`.

### `.sdlc/pm-log.md` (auto-maintained)
Persistent record of PM cycle runs: what was audited, what the competitive landscape looked like, what opportunities were identified and why. Allows the next PM run to see what has changed.

---

## Components

### Agents

#### `product-auditor`
Reads the current codebase and the SDLC decision log. Produces a structured assessment of:
- What is currently built (feature inventory)
- What is working well (from decision log patterns, test results)
- What is incomplete, brittle, or missing
- How the current state measures against `PRODUCT.md` goals and metrics
- Gaps between stated goals and implemented reality

Does not use web search. Reads only local state.

#### `competitive-scanner`
Takes the product domain and current feature inventory from the audit. Uses `WebSearch` and `WebFetch` to:
- Map 3–5 comparable products or tools in the same space
- Identify capabilities they have that the current product lacks
- Identify differentiators the current product has
- Surface trends or emerging patterns in the space

Produces a structured competitive landscape report.

#### `opportunity-synthesizer`
Cross-references the audit + competitive scan + `PRODUCT.md` goals. Produces:
- **Gap opportunities:** things competitors have that align with stated goals
- **Improvement opportunities:** things that exist but could be significantly better
- **Moonshot opportunities:** hypothetical ideas unconstrained by current scope — what could this become?

Ranks opportunities by estimated impact vs. effort. Flags which are incremental vs. transformational.

#### `brief-writer`
Takes one approved opportunity from the synthesizer. Produces a complete Product Brief:
- Opportunity statement
- Evidence (from audit + competitive scan)
- Success criteria (structured as testable outcomes — feeds directly into SDLC spec ACs)
- Scope boundaries (what this brief does and does not cover)
- Moonshot context if applicable
- Suggested blueprint type for SDLC

### Templates

#### `product-brief.md`
The handoff artifact. Structured enough that SDLC Phase 1 can bootstrap a spec directly from it.

#### `competitive-landscape.md`
Output of the competitive-scanner. Persisted to `.sdlc/pm-log.md` for the next cycle.

#### `pm-cycle-summary.md`
Summary written at the end of each PM run: what was found, what briefs were produced, what was deferred.

---

## SDLC Integration

### Phase 1 change (SDLC skill update required)
When `$ARGUMENTS` is a path to a `.sdlc/briefs/{{brief}}.md` file, SDLC Phase 1 reads the brief instead of prompting for requirements. The spec-writer agent bootstraps:
- Functional requirements from the brief's opportunity statement
- Acceptance criteria from the brief's success criteria
- Out-of-scope from the brief's scope boundaries
- NFRs from `PRODUCT.md` metrics that apply to this feature type

This means a brief-to-spec translation is largely automated — the human's approval of the brief IS the requirements sign-off.

### Blueprint hint
The Product Brief includes a `suggested-blueprint` field. SDLC Phase 2 uses this as a starting point for blueprint selection rather than inferring from scratch.

---

## Autonomy Model

| Step | Autonomous | Human gate |
|------|-----------|------------|
| Codebase audit | ✓ | |
| Competitive scan | ✓ | |
| Opportunity synthesis | ✓ | |
| Brief generation | ✓ | |
| Brief approval | | ✓ Review queue |
| SDLC execution | ✓ (per-phase gates) | ✓ Spec + arch approval |

The PM skill can run on a schedule via `/schedule` — weekly or monthly — depositing briefs into the queue. A human reviews the queue and approves. SDLC execution follows the existing human-gated phase structure.

For low-risk opportunity types (documentation, minor UX improvements, test coverage), the approval gate could be bypassed with explicit configuration. For architectural changes, the gate is non-negotiable.

---

## Build Approach

**Use `/sdlc` to build the PM skill.** Reasons:

1. **Dogfooding** — the best way to validate and improve the SDLC skill is to use it on a real, non-trivial feature
2. **No existing blueprint** — building a Claude Code plugin is its own pattern type. Running `/sdlc` against this will expose the gap and drive creation of a `claude-code-plugin` blueprint — which is itself valuable for all future plugin development
3. **Structured output** — the SDLC skill will produce a spec, arch doc, and contract for the PM skill, which is better planning than ad-hoc implementation

The input to `/sdlc` would be a path to this plan file, or this plan file would be used to manually write the first Product Brief.

**Pre-requisite:** a `claude-code-plugin` blueprint needs to exist before running `/sdlc` on this. Otherwise Phase 2 falls through to no-bootstrap mode and the arch-designer works without a pattern template. Writing that blueprint is the first concrete next step.

---

## Open Questions

1. **Scheduling:** should the PM skill auto-run on a schedule by default, or only on-demand? An auto-schedule risks noise; on-demand risks never running.
2. **Competitive scan scope:** how broad? Whole product, or per-feature-area? A full product scan is expensive; per-area is more actionable but requires the user to frame the question.
3. **Moonshot quality:** moonshot generation is the hardest to do well. What grounding prevents the synthesizer from generating ideas that are irrelevant to the product domain?
4. **PRODUCT.md bootstrapping:** how does the user create a good `PRODUCT.md`? Should there be a one-time setup flow that helps them write it?
5. **Brief prioritization:** who ranks the brief queue — the synthesizer or the human? Synthesizer ranking is cheap but may not match business priorities.

---

## Next Steps

1. Write the `claude-code-plugin` blueprint
2. Run `/sdlc` against this plan to produce the PM skill spec and architecture
3. Review the arch doc — does it match this plan's component design?
4. Proceed through SDLC phases to implement
