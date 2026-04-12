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

| File | Update when |
|---|---|
| PROJECT.md | Rarely — only on fundamental vision change |
| REQUIREMENTS.md | New brainstorm output, scope change |
| ROADMAP.md | Milestone complete, new milestone starts |
| STATE.md | After each step completes, when new blockers arise |
| {phase}-RESEARCH.md | After Step 4 (Research) |
| {phase}-{N}-PLAN.md | After Step 6 (Planning) |
| {phase}-SUMMARY.md | After Step 11 (Ship) |
| {phase}-UAT.md | After Step 8 (UAT) |

## Templates

See `docs/claude/templates/` for each file's template.

## Commit Behavior (`commit_docs` config)

`commit_docs` in `.planning/config.json` controls what gets committed to git:

| Setting | Default | Scope |
|---|---|---|
| `state_files` | `true` | PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md |
| `planning_artifacts` | `true` | `.planning/` — research, plans, UAT, summaries, verification |

When `planning_artifacts: false`:
- `.planning/` is still created on disk (artifact persistence preserved)
- `.planning/` is auto-added to `.gitignore`
- State files (PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md) still committed normally
- Resume still works because STATE.md remains in git

## Principles

- STATE.md is the single source of truth for "where are we"
- Do not store code patterns, git history, or debug solutions in state files
- Use absolute dates, not relative ("2026-04-08" not "today")
- STATE.md must be sufficient for a new session to know exactly what to do next