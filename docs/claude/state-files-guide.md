# State Files Guide

State files are the project's "real memory" — they live in the repo, are readable by both AI and humans, and persist across context resets.

## Structure

```
{project-root}/
├── PROJECT.md          — Vision document, stable within a milestone
├── REQUIREMENTS.md     — Scope v1/v2 with phase traceability
├── ROADMAP.md          — Milestone list + progress
├── STATE.md            — Current position, decisions, blockers
└── .planning/
    ├── {phase}-RESEARCH.md      — Research output per phase
    ├── {phase}-{N}-PLAN.md      — Plan per task group (XML structure)
    ├── {phase}-SUMMARY.md       — Record of what happened
    └── {phase}-UAT.md           — User acceptance test results
```

## When to Create

- **New project (Step 0)**: Create PROJECT.md + REQUIREMENTS.md + ROADMAP.md + STATE.md
- **Brownfield project (Step 0)**: Map codebase first → create state files from mapping results
- **Resuming session**: Read STATE.md → immediately know phase, blockers, next action

## When to Update

| File | Update when | Update owner |
|---|---|---|
| PROJECT.md | Rarely — only on fundamental vision change | User |
| REQUIREMENTS.md | New brainstorm output, scope change | brainstorming skill |
| ROADMAP.md | Milestone complete, new milestone starts | writing-plans + finishing-a-development-branch skills |
| STATE.md | After each step completes, when new blockers arise | every major skill (see below) |
| {phase}-RESEARCH.md | After Step 4 (Research) | Research agents |
| {phase}-{N}-PLAN.md | After Step 6 (Planning) | writing-plans skill |
| {phase}-SUMMARY.md | After Step 11 (Ship) | finishing-a-development-branch skill |
| {phase}-UAT.md | After Step 8 (UAT) | User-driven |

### STATE.md update chain (Layer 1)

Each major workflow skill has a mandatory "Update STATE.md" sub-step at its end:

| Skill | STATE.md fields it updates |
|---|---|
| `brainstorming` | Phase, Status, Last updated, Next Action, Approved Mode, Key Decisions |
| `writing-plans` | Phase, Status, Last updated, Next Action, Key Decisions; also updates ROADMAP.md |
| `executing-plans` | Phase, Status, Last updated, Next Action, Open Blockers — **after each wave** |
| `subagent-driven-development` | Phase, Status, Last updated, Next Action, Open Blockers — **after each task** |
| `finishing-a-development-branch` | Phase, Status, Last updated, Next Action, Current Milestone; also updates ROADMAP.md + writes `.planning/{phase}-SUMMARY.md` |

If any skill exits without updating STATE.md, validate-state will catch it on next session.

### Validation (Layer 2)

The `validate-state` skill is the safety net. It runs:

- **Mandatory at STEP 0 (every session resume)** — blocks on FAIL, asks user on WARN
- **Recommended before STEP 7 (Execute)** — avoid working from stale state
- **On-demand** when user says "validate state" or "check state"

It performs 4 categories of checks: Existence, Schema, Freshness, Drift. See `skills/validate-state/SKILL.md` for full spec.

### Cross-platform safety net (Layer 3)

The plugin's SessionStart hook (Claude Code + Cursor + Copilot CLI) detects when `STATE.md` exists in the project root and injects a reminder to run validate-state. On harnesses without hooks (Codex), the CLAUDE.md STEP 0 directive ensures the same behavior.

## Templates

See `docs/claude/templates/` for each file's template.

## Commit Behavior

What gets committed is controlled by `commit_docs` in `.planning/config.json` — see `config-schema.md` for the flags. Key point for state files: even with `planning_artifacts: false` (which gitignores `.planning/`), the state files (PROJECT / REQUIREMENTS / ROADMAP / STATE.md) are still committed — so **resume always works**, because STATE.md stays in git.

## Principles

- STATE.md is the single source of truth for "where are we"
- Do not store code patterns, git history, or debug solutions in state files
- Use absolute dates, not relative ("2026-04-08" not "today")
- STATE.md must be sufficient for a new session to know exactly what to do next