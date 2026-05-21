---
name: fast-lane-assessment-v1
description: Use at STEP 1 before brainstorm to decide if a task qualifies for Fast Lane (skip brainstorm + mode gate + research). Evaluates scope, surface, architectural impact, security/migration risk, and regression risk against criteria from current-process-workflow.md.
---

# Fast Lane Assessment

## Overview

Decide whether the current task qualifies for the Fast Lane path. If eligible, STEPS 2-4 (brainstorm, mode selection, research) are skipped and the workflow goes directly to STEP 5 (Spec).

**Core principle:** Conservative gate — when in doubt, run the full workflow. Fast Lane is for genuinely small, isolated, low-risk work only.

**Announce at start:** "I'm using the fast-lane-assessment-v1 skill to check Fast Lane eligibility."

## Sources of Truth

This skill does not redefine criteria or output schema. It references:

- **Eligibility criteria:** `docs/claude/current-process-workflow.md` → STEP 1
- **Output template:** `docs/claude/agent-output-templates.md` → T6 (`fast-lane-assessment-v1`)

Read both before running the assessment.

## The Process

### Step 1: Score the Five Criteria

Score the task against each criterion from STEP 1 of the workflow. All five must be true to be eligible.

| # | Criterion | Pass condition |
|---|---|---|
| 1 | Fix scope is clear and unambiguous | Single behavior to change, requirements stated explicitly |
| 2 | Small change | 1-2 files modified |
| 3 | No architecture/API contract change | No new exports, no signature change, no schema change |
| 4 | No security or data migration impact | No auth, no PII, no migration scripts |
| 5 | Low regression risk | Touch surface used by few callers, or has tests |

**Hard exclusions (any one disqualifies):**
- Cross-cutting concern (touches >1 module)
- Refactor or rename across files
- New feature (even small)
- Any user-facing behavior change beyond fixing a defect
- Anything blocking a release

### Step 2: Emit the Assessment Output

Produce JSON matching template T6 in `docs/claude/agent-output-templates.md`. Required fields:

- `eligible` (bool) — true only if all five criteria pass and no hard exclusion applies
- `task_class` — `hotfix` / `small_task` / `normal`
- `analysis` — one boolean per criterion, mirroring the table
- `recommended_skips` — usually `["brainstorm"]` when eligible
- `required_minimum_gates` — always include `verification` (STEP 8/9 cannot be skipped)
- `reason` — one sentence justifying the verdict
- `fallback_path` — what to do if a risk factor surfaces mid-execution

Save the output inline in the response, or persist to `.planning/fast-lane-<task-slug>.json` if `commit_docs.planning_artifacts: true`.

### Step 3: Present and Wait for User Confirmation

Show the verdict + reason. Ask:

```
Fast Lane eligible: <yes/no>
Reason: <one sentence>

Proceed Fast Lane (skip to STEP 5), or run full workflow?
```

**Never auto-advance.** User confirms the path.

## Outputs Per Verdict

**Eligible (Fast Lane):**
- Skip STEP 2 (brainstorm), STEP 3 (mode gate), STEP 4 (research)
- Go to STEP 5 (Spec) with the assessment JSON as input artifact
- STEP 6 plan may still be required if 2+ tasks emerge; otherwise inline in STEP 7

**Not eligible (full workflow):**
- Continue to STEP 2 (brainstorm)

## Red Flags

**Never:**
- Mark eligible when uncertain — default to not eligible
- Skip the assessment to save time on "obviously simple" tasks (the rationalization itself is a signal)
- Fast Lane anything touching `auth*`, `migration*`, `release*`, `*-config*`, or schema files
- Fast Lane a task you cannot describe in one sentence

**Always:**
- Run this check before STEP 2 on every task
- Get explicit user confirmation of the verdict
- Document the verdict (inline or persisted) so STEP 8 verification can cross-check

## Integration

**Called by:** controller at STEP 1 of `docs/claude/current-process-workflow.md`

**Inputs:**
- Task description from user
- Project state from STATE.md, REQUIREMENTS.md
- `.planning/config.json` (for `commit_docs.planning_artifacts` flag)

**Outputs:**
- Assessment JSON (template T6)
- User-confirmed path: Fast Lane or full workflow

**Pairs with:**
- `skills/brainstorming/SKILL.md` — invoked if not eligible
- `skills/writing-plans/SKILL.md` — invoked at STEP 6 regardless
- `skills/verification-before-completion/SKILL.md` — invoked at STEP 8/9 regardless of path
