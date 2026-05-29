---
artifact: bootstrap
name: claude-code-plugin
version: 1
derived-from: plugins/pm (PM Skill, 2026-05-29)
tags: [plugin, skill, claude-code, markdown, agent]
---

# Bootstrap: Claude Code Plugin

## When to Use

Apply this bootstrap when building a feature that:
- Adds a new skill (slash command) to Claude Code via the plugin system
- Requires one or more sub-agents invoked by an orchestrating SKILL.md
- Produces or transforms persistent state files (not application code)
- Is packaged as a plugin in `plugins/<name>/` and registered via `.claude-plugin/marketplace.json`

Do not apply when: the feature is a code library, a REST API, a background worker, or any component that runs as a server or service. Those features live inside an application codebase, not in the plugin directory.

---

## Canonical Component Graph

```
C1 — Skill Orchestrator (SKILL.md)
  │  Owns: entry point, argument parsing, pipeline sequencing, state reads/writes
  │  Does not own: domain logic, web research, file template structure
  │
  ├──▶ C2..Cn — Domain Agents (agents/*.md)
  │           Owns: one bounded task each (analysis, research, synthesis, generation)
  │           Does not own: orchestration, file writes (except designated writer agent)
  │
  └──▶ Cx — Template(s) (templates/*.md)
             Owns: canonical output structure consumed by downstream skills or agents
             Does not own: logic or state
```

Plus two infrastructure files:
- **C_manifest** — `.claude-plugin/plugin.json` — declares plugin identity
- **INT_marketplace** — `.claude-plugin/marketplace.json` — registers plugin with loader (modify existing)

Dependencies flow from C1 downward. Agents do not depend on each other directly — C1 passes outputs between them.

---

## Required Interfaces (abstract)

**C1 — Skill Orchestrator (SKILL.md)**
```yaml
# frontmatter
name: <plugin-name>
description: <one sentence>
argument-hint: <CLI argument pattern>
allowed-tools: [Read, Write, Edit, Glob, Grep, TodoWrite, Agent]
```
Body must contain:
- `## Standing Context` — files read at startup, held in working memory
- `## Startup Checks` — pre-pipeline gates (e.g., required files, optional config reads)
- `## Pipeline` — numbered steps, one per agent, with explicit inputs/outputs
- `## Post-Pipeline Writes` — all file mutations, paths, and conditions
- `## Permitted Write Targets` — explicit table of allowed write paths; all others read-only

**Cn — Domain Agent (agents/*.md)**
```yaml
# frontmatter
name: <agent-name>
description: <one sentence>
tools: <comma-separated list — include only what this agent needs>
model: sonnet
```
Body must contain:
- `## Inputs` — what the invoking skill passes in
- `## Process` — step-by-step instructions
- `## Output Format` — exact structure returned to the orchestrator
- `## Constraints` — explicit prohibitions (no web search, no file writes, etc.)

Tool list is the primary permission control. Do not add tools an agent does not need:
- Read-only agents: `Read, Grep, Glob` (or `Read` alone)
- Web research agents: `Read, WebSearch, WebFetch` (no Write)
- File-writing agents: `Read, Write, Glob` (no web tools unless required)

**Cx — Template (templates/*.md)**
No frontmatter required. The template file defines the output scaffold that a writing agent fills at runtime. Each placeholder must make the field's requirement unambiguous. Required sections must be annotated as required; optional sections must be annotated as optional.

**C_manifest — plugin.json**
```json
{
  "name": "<plugin-name>",
  "version": "0.1.0",
  "description": "<one sentence>",
  "author": { "name": "<name>", "email": "<email>" }
}
```

**INT_marketplace — marketplace.json (modify)**
Add one entry to the `plugins` array:
```json
{
  "name": "<plugin-name>",
  "description": "<one sentence>",
  "author": { "name": "<name>" },
  "category": "development",
  "source": {
    "source": "git-subdir",
    "url": "file:///<path-to-repo>",
    "path": "plugins/<plugin-name>",
    "ref": "master"
  }
}
```

---

## Directory Layout

```
plugins/<name>/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── <name>/
│       └── SKILL.md
├── agents/
│   ├── <agent-1>.md
│   └── <agent-n>.md
└── templates/
    └── <template>.md
```

The skill loader discovers `skills/<name>/SKILL.md` by convention. The `name` in `plugin.json` must match the directory name.

---

## Adaptation Questions

Survey the codebase to answer these before designing the component graph:

1. **What is the entry argument pattern?** Does the skill take free-text, structured arguments (key=value), or a file path?
2. **What persistent state does the skill read?** Are there required config files (like PRODUCT.md) or optional state files (like pm-ideas.md)?
3. **What persistent state does the skill write?** List every output path. Define the permitted write targets before designing agents.
4. **How many agents are needed?** One agent per bounded domain task. If a task requires both web research and file writes, split it into two agents.
5. **Which agents need web access?** Only agents that need to call external services should have WebSearch/WebFetch in their tools list.
6. **Which agents write files?** Prefer having only the orchestrator and one designated "writer" agent write files. This makes NF3-style write isolation easier to enforce.
7. **Is there a downstream consumer of the output?** If another skill (like SDLC) reads this skill's output, the template structure must be designed to match the consumer's input format.

---

## Known Anti-Patterns

- **Monolithic SKILL.md:** putting all logic in one prompt file. Breaks at non-trivial codebase sizes and makes the skill untestable phase-by-phase. Use sub-agents.
- **Agents that write files directly:** all file mutations should flow through C1 or a single designated writer agent. Dispersed writes make NF3 enforcement (permitted-write-targets) impossible to verify.
- **Missing tool restrictions:** giving all agents the full tool set. The tool list is the permission model — every unnecessary tool is a potential for scope violations. Each agent should have only the tools it genuinely needs.
- **Parallel agents with data dependencies:** do not invoke agents in parallel if one needs the output of another. The data dependency must be resolved first.
- **Implicit output format:** agents returning unstructured text. Always define an `## Output Format` section with named fields so the orchestrator can reliably parse and pass results downstream.
