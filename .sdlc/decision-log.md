# Decision Log

---

## SDLC ↔ PM Integration — 2026-05-29

### D8 — SDLC Phase 1 accepts approved Product Briefs as input
**Decision:** When `$ARGUMENTS` is a path to a `.sdlc/briefs/*.md` file, SDLC Phase 1 reads the brief and bootstraps the spec from it (opportunity statement → requirements, success criteria → ACs, scope-out → Out of Scope, suggested-blueprint → Phase 2 hint). Briefs with `status` other than `approved` are rejected with a clear message.
**Rationale:** Closes the PM→SDLC loop. Brief approval by the human is the requirements sign-off — the spec can be largely auto-generated from it, reducing the human-in-the-loop cost of starting an SDLC run.

### D9 — Brief approval is a manual frontmatter edit
**Decision:** To approve a brief, the human edits `status: pending` → `status: approved` in the brief file. No separate approval command is introduced.
**Rationale:** Keeps the system simple. The human's intentional edit is the gate. No new skill or tool is needed.

---

## PM Skill — 2026-05-29

### D1 — Plugin structure follows SDLC plugin convention
**Decision:** The PM skill uses the same directory layout as the SDLC plugin: `.claude-plugin/plugin.json`, `skills/<name>/SKILL.md`, `agents/*.md`, `templates/*.md`. All components are markdown or JSON files — no code.
**Rationale:** The SDLC plugin was the only reference implementation available. Following its convention exactly ensures the Claude Code plugin loader can discover the PM skill without any loader changes. It also establishes a repeatable pattern for all future plugins.

### D2 — Sequential agent pipeline (not parallel)
**Decision:** The PM pipeline runs C2 → C3 → C4 sequentially.
**Rationale:** C3 (competitive-scanner) requires C2's feature-inventory to scope its search meaningfully. C4 (opportunity-synthesizer) requires both C2 and C3 outputs. The data dependencies make parallel execution incorrect, not just inconvenient.

### D3 — Soft moonshot filter (discarded-moonshots section)
**Decision:** Moonshots that fail the PRODUCT.md goal-citation gate appear in a `discarded-moonshots` section rather than being silently dropped.
**Rationale:** User preference — seeing what was filtered and why is more useful than invisible discard. Allows users to uncover cases where PRODUCT.md goals are too narrowly stated.

### D4 — Human ideas via checkbox file (.sdlc/pm-ideas.md)
**Decision:** Human-provided ideas are read from `.sdlc/pm-ideas.md` using markdown checkbox syntax. Unchecked items (`- [ ]`) are evaluated; checked items (`- [x]`) are skipped. After evaluation, the skill marks ideas as checked.
**Rationale:** Avoids re-evaluation cost across runs. The human can uncheck an idea to force re-evaluation. Works with scheduling (ideas can be queued anytime, not just at run time). No interactive pause required mid-pipeline.

### D5 — No persistent competitive-landscape file
**Decision:** Competitive scan results are not persisted as a separate file between runs. They are captured in the pm-log.md cycle summary only.
**Rationale:** A separate file would require an additional write path beyond the four permitted by NF3. The pm-log.md provides sufficient historical record for trend analysis across cycles.

### D6 — PM skill is on-demand only
**Decision:** The PM skill does not set up its own cron schedule. Users who want periodic runs use the `/schedule` skill separately.
**Rationale:** Auto-scheduling risks noise; on-demand keeps the human in control of when PM analysis runs. The `/schedule` skill already provides scheduling infrastructure.

### D7 — No claude-code-plugin blueprint existed; this feature is the candidate
**Decision:** Architecture was designed without a bootstrap by surveying the SDLC plugin as the reference pattern. A `claude-code-plugin` blueprint should be created using this feature as the source.
**Rationale:** No blueprint existed for this feature type. The SDLC plugin provided the closest reference. After implementation, this feature's pattern is well-understood enough to codify as a blueprint.
