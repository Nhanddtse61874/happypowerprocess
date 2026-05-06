# /init-project + /update-config — Phase Summary

**Phase:** init-project (plugin self-improvement)
**Completed:** 2026-05-06
**Mode:** A (Solo)
**Released as:** v5.4.0

## What Was Built

### New skills
- **`/init-project`** ([skills/init-project/SKILL.md](../skills/init-project/SKILL.md)) — bootstrap project state files, config, CLAUDE.md, permissions, gitignore. 2 minimal prompts (project name + primary stack), idempotency 4-choice menu, `--force` flag with backup, brownfield auto-detection via Glob markers, cross-platform plugin path detection with Glob fallback.
- **`/update-config`** ([skills/update-config/SKILL.md](../skills/update-config/SKILL.md)) — interactive config update, per-field validation against schema, mid-workflow impact warnings, auto `.gitignore` management on `commit_docs.planning_artifacts` toggle.

### Supporting artifacts
- **[docs/claude/config-schema.md](../docs/claude/config-schema.md)** — single source of truth for 9 config fields (12 rendered rows), display grouping convention, Read-permission trade-off documentation.
- **[docs/claude/templates/CLAUDE.md](../docs/claude/templates/CLAUDE.md)** — project-level CLAUDE.md template (consumed by `/init-project`).
- **[docs/claude/templates/settings.json](../docs/claude/templates/settings.json)** — Option C workflow-aware permission allowlist.
- **[docs/claude/templates/gitignore](../docs/claude/templates/gitignore)** — base gitignore entries.
- **[commands/init-project.md](../commands/init-project.md)** + **[commands/update-config.md](../commands/update-config.md)** — slash command registrations for cross-harness compatibility.

### Plugin integration
- Plugin manifests: 4 new keywords (init-project, bootstrap, config-management, project-setup) in both `.claude-plugin/plugin.json` + `.cursor-plugin/plugin.json`.
- Plugin CLAUDE.md: new "Bootstrap Commands (New in v5.4.0)" section.
- Version bump 5.3.0 → 5.4.0 across 5 manifests + CHANGELOG entry.

### Documentation
- **[docs/specs/2026-05-06-init-project-skill-design.md](../docs/specs/2026-05-06-init-project-skill-design.md)** — full design spec (Problem → Goals → Components → Data Flow → Error Handling → Testing → Decisions Log).
- 3 implementation plans in XML format ([init-project-01-PLAN.md](init-project-01-PLAN.md), [init-project-02-PLAN.md](init-project-02-PLAN.md), [init-project-03-PLAN.md](init-project-03-PLAN.md)).

## Key Decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | Option C init scope (mechanical + 2 prompts) | Match user expectation, balance speed/depth. Vision/REQ-IDs deferred to STEP 2 brainstorm. |
| 2 | Option B update-config UX (walk subset) | Speed when only changing 1-2 fields, validate per-field. |
| 3 | Option B idempotency 4-choice menu | Cover all re-run scenarios safely. `--force` flag for power user. |
| 4 | Option B brownfield (stack detection only) | Avoid overlap with brainstorm; full architecture mapping is premature. |
| 5 | A2+B3 git/commit prompts | Explicit before side-effects; user mới hiểu rõ AI làm gì. |
| 6 | Option C permissions (workflow-aware defaults) | Smooth read-heavy ops, still safe for write/destructive. |
| 7 | Approach 2 shared schema doc | Single source of truth, future-proof when schema gains fields. |
| 8 | Mode A (Solo) | 0/5 Mode B signals (single domain, low risk, no formal QA gate). |
| 9 | Skip STEP 4 Research | Pattern known from existing /version skill, no novel domain knowledge needed. |
| 10 | Subagent-driven execution + atomic commits + no worktrees | Fresh context per task, bisectable history, sequential within wave. |
| 11 | Drop name "Superpowers" from project-level CLAUDE.md template | User building "happypowerprocess" — own branding. |
| 12 | State files in `.planning/` (not root) | User preference for cleaner project root. |

## Commits (chronological)

### Planning (1)
- `5a22161` chore(init-project): planning artifacts for /init-project + /update-config skills

### Wave 1 — Foundation files (3)
- `5cf2a78` feat(init-project): create config-schema.md (single source of truth)
- `b22d6d9` feat(init-project): create settings.json permission allowlist template
- `6970f03` feat(init-project): create gitignore template

### Wave 2 — Main skills (2)
- `c4d622d` feat(init-project): create init-project skill for project bootstrap
- `d027430` feat(init-project): create update-config skill for interactive config edit

### Wave 3 — Plugin integration (2)
- `2e75ea1` feat(init-project): add init-project + update-config keywords to plugin manifests
- `75c73f7` feat(init-project): document /init-project + /update-config in plugin CLAUDE.md

### QA fixes round 1 (5)
- `9414ef5` fix(init-project): address QA findings I3, I6, I7, S3 in init-project skill
- `01af74e` fix(init-project): address QA findings I1, I2, I5, I6 in update-config skill
- `a72ba81` fix(init-project): simplify gitignore template per QA finding I4
- `1b1413d` fix(init-project): document grouping convention + read permission note (I2, S4)
- `50e0265` fix(init-project): add commands/init-project.md + commands/update-config.md (C1)

### QA fixes round 2 (1)
- `0382fb5` fix(init-project): clean up stale references after gitignore simplification (NEW-I1, NEW-S1)

### Release (1)
- `90a7352` chore: release v5.4.0 (init-project + update-config skills)

**Total**: 16 commits.

## What Worked

- **11-step workflow ceremony** drove discipline: brainstorm → mode gate → spec → plan → execute → QA → release. Each step had clear handoff and user touchpoint.
- **XML plans with frontmatter `must_haves`** made Goal-Backward Verification mechanical — cross-reference artifacts with truths/key_links was straightforward.
- **Subagent-driven execution** with model overrides: user could escalate from haiku → sonnet → opus per task. Sonnet sufficient for most, opus for the 2 main skill files.
- **Atomic commits** with REQ-ID + plan path in commit message body — easy bisect, easy audit trail.
- **Single source of truth (config-schema.md)**: when QA found `1-10` hardcoded range was brittle, fix was straightforward — schema absorbs the "Display Grouping Convention" and skill becomes dynamic.
- **Cross-skill consistency** for shared concerns (plugin path detection): both skills got identical Glob fallback in same QA round — same pattern, same wording.
- **QA gate with code-reviewer subagent** caught real issues:
  - C1 missing command files (would have shipped non-functional skills if missed)
  - I1 hardcoded examples (drift risk over time)
  - I5 mtime check (out-of-scope per spec but slipped into implementation)

## What Didn't Work / Lessons

- **Hardcoded literal text in skills** is a recurring trap. Both skills initially baked schema content (mid-workflow impact text, valid value lists, index counts) directly. QA found this in 3 places (I1, I2). **Lesson**: when schema is single source of truth, skills should ALWAYS defer to schema at runtime, never copy-paste schema content.
- **mtime concurrent-edit check** was over-engineering for "single-user repo assumed" out-of-scope item. Skill author added it instinctively (defensive coding), but QA correctly flagged it as inconsistent with spec scope. **Lesson**: re-check spec out-of-scope items before adding "just in case" features.
- **gitignore template comment block** caused cascade of "do not touch this comment line" carve-outs in 2 skills. After simplifying template, 2 skill files needed cleanup (NEW-I1). **Lesson**: minimize "negative" instructions in skills; remove the source of confusion rather than protect it.
- **Marketplace version drift** (5.2.0 vs 5.3.0) was pre-existing and silent. Bump-version.sh has `--check` for drift detection but jq missing on Windows. **Lesson**: drift checking should be part of CI, not depend on local jq.

## Residual Risks

| # | Risk | Mitigation |
|---|---|---|
| R1 | Plugin path detection still fails for unusual installs (vendored, symlinked) despite Glob fallback | Add install-time validation; print resolved plugin path on first run for debugging |
| R2 | `gitignore` template name (no leading dot) is unusual — future maintainer may "fix" it by renaming to `.gitignore` (which would self-gitignore) | Add comment header in template explaining naming choice. *Deferred to follow-up.* |
| R3 | Schema doc has no programmatic validation regex — relies on AI interpreting "Valid values" column | Future: add regex column or JSON schema sidecar |
| R4 | STATE.md "Notes" section grows unbounded over time | Future: log rotation feature (cap at last N entries) |
| R5 | jq missing on Windows breaks `bump-version.sh --check` | Use PowerShell port of script, or document `winget install jq` requirement |

## Next Milestone Input

Ideas for v5.5.0+ (collected during this work):

1. **Hooks integration**: SessionStart hook detects "no STATE.md" and offers `/init-project` (per Hướng C of original brainstorm).
2. **STATE.md log rotation**: cap "Notes" section at last N entries to prevent unbounded growth.
3. **Pre-commit hook validator**: ensure `commit_docs` config matches actual git status (no `.planning/` accidentally committed when `planning_artifacts: false`).
4. **Stack-specific permission templates**: extend `/init-project` to add stack-specific `Bash(dotnet:*)` / `Bash(npm:*)` to settings.json based on detected stack (was Option D in Q6, deferred).
5. **Config schema regex**: add validation regex column to `config-schema.md` for stricter value checking.
6. **Multi-team architecture exploration**: brainstorm session noted that for very large projects, splitting work across 2+ team-orchestrators with shared meta-orchestrator could 1.5-1.8× throughput. Out of scope for v5.4.0; possibly v6.0.

## Workflow Compliance Check

| Step | Status | Notes |
|---|---|---|
| 0. Bootstrap | N/A | Plugin self-dev — no formal `.planning/` init |
| 1. Fast Lane | ❌ Not eligible | Multi-file change |
| 2. Brainstorm | ✅ | 6 clarifying questions + 5-section design + spec |
| 3. Mode Selection Gate | ✅ | Mode A (0/5 Mode B signals) |
| 4. Research | ⏭ Skipped | User decision (Option A) — pattern known from /version skill |
| 5. Spec | ✅ | docs/specs/2026-05-06-init-project-skill-design.md |
| 6. Plan | ✅ | 3 XML plans, goal-backward, must_haves frontmatter |
| 7. Execute | ✅ | 7 tasks across 3 waves, subagent-driven, atomic commits, model overrides per user |
| 8. UAT + Verification | ⏭ UAT skipped (user Option B) | Goal-Backward Verification PASS via must_haves cross-ref |
| 9. QA Gate | ✅ | 2 rounds: BLOCK → APPROVE WITH CONDITIONS → APPROVE |
| 10. Release/DevOps | ✅ Lightweight | Version bump + CHANGELOG (no full skill ceremony) |
| 11. Ship | ✅ | This summary + push to remote |

---

**End of phase summary.**
