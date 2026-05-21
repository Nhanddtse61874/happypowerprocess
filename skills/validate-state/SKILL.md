---
name: validate-state
description: Use at STEP 0 (session resume) and on-demand when state may have drifted. Validates that state files (PROJECT/STATE/REQUIREMENTS/ROADMAP) exist, follow schema, are fresh against latest artifacts, and are not contradicted by .planning/ contents. Catches drift before AI acts on stale state.
---

# Validate State

## Overview

State files are the project's persistent memory. They are updated by humans and AI across sessions, and can drift from reality when an update is missed (AI crashed mid-wave, human edited file, branch merged without state update). Stale state silently misleads every downstream decision — resume continues from the wrong step, REQ-ID verification false-passes, milestone progress is wrong.

This skill is the **safety net**: read the state files, the `.planning/` artifacts, and recent git activity, compare them, and report drift before any other step proceeds.

**Core principle:** Trust nothing about state until verified against artifacts on disk.

**Announce at start:** "I'm using the validate-state skill to check state files for drift."

## When to Invoke

| Trigger | Behavior |
|---|---|
| STEP 0 of every session (resume) | **MANDATORY** — block until clean or user accepts drift |
| Before STEP 7 (Execute) | Recommended — avoid working from stale state |
| After unexpected interruption (crash, network drop) | On-demand |
| User says "validate state" or "check state" | On-demand |
| Mid-wave when something feels off | On-demand |

## What to Check — 4 Categories

### Category 1: Existence

Required files per `docs/claude/state-files-guide.md`:

- `PROJECT.md` — vision (rarely changes)
- `STATE.md` — current position (changes most often)
- `REQUIREMENTS.md` — REQ-IDs and v1/v2 scope
- `ROADMAP.md` — milestones

Optional but informative:
- `.planning/config.json` — workflow config
- `.planning/{phase}-{N}-PLAN.md` — plan artifacts
- `.planning/{phase}-SUMMARY.md` — completed phase summaries

**Output**: list missing required files. If any missing → **FAIL** (project not initialized — direct user to `/init-project`).

### Category 2: Schema Validation

For each existing state file, verify required sections + format.

**STATE.md required sections + format:**

```
## Current Position
**Phase:** Step <N> — <step name>
**Status:** <in_progress | blocked | waiting_for_user | complete>
**Last updated:** <YYYY-MM-DD>

## Next Action
<one sentence>

## Approved Mode
<Mode A | Mode B> — approved <YYYY-MM-DD>
```

Checks:
- `Status` value is one of the enum
- All dates match `YYYY-MM-DD` (regex `^\d{4}-\d{2}-\d{2}$`)
- Phase number is integer 0-11
- `Next Action` is non-empty
- `Approved Mode` is `Mode A` or `Mode B`

**REQUIREMENTS.md schema:**

- Each REQ-ID matches `^[A-Z]{2,8}-\d{2,4}$` (e.g., `AUTH-01`, `CONTACT-02`)
- v1 table has columns: `REQ-ID | Requirement | Phase`
- Each v1 REQ-ID has a Phase assignment (not empty)
- No duplicate REQ-IDs

**ROADMAP.md schema:**

- `Active Milestone` section exists with non-empty milestone name + status enum
- `Last updated` date matches `YYYY-MM-DD`
- Phase count consistent with `config.json` granularity (coarse=3-5 / standard=5-8 / fine=8-12+)

**PROJECT.md schema:**

- `Vision`, `Problem Statement`, `Success Criteria`, `Stack` sections all present and non-empty
- `Created` date matches `YYYY-MM-DD`

**Output**: list schema errors per file. If any error → **FAIL** (block until fixed).

### Category 3: Freshness

Compare `STATE.md` "Last updated" date with:

1. **Latest `.planning/` artifact mtime** — newest of `*.md` files in `.planning/`
2. **Latest commit date** affecting files NOT in `state_files` list (i.e., code changes, not state file commits)

**Drift signals:**

- `STATE.md last_updated` is older than latest `.planning/` artifact by ≥1 day → likely missed update after plan/summary write
- `STATE.md last_updated` is older than latest non-state-file commit by ≥1 day → likely missed update after code commit
- `ROADMAP.md last_updated` is older than 30 days while there is active work in STATE.md → ROADMAP stale

**Output**: list freshness warnings. **WARN** level — does not block, but must report and let user accept or fix.

### Category 4: Drift Detection (Cross-Reference)

Compare claims across state files and artifacts:

**Check D1: Step claim vs artifact existence**

- STATE.md says `current_step: Step 6` AND `.planning/{phase}-SUMMARY.md` exists → contradiction. Plan SUMMARY means phase complete, but state says still in Plan step.
- STATE.md says `current_step: Step 7` AND no `.planning/{phase}-*-PLAN.md` exists → contradiction. Execute step can't run without a plan.

**Check D2: REQ-ID phase coverage**

For each REQ-ID in REQUIREMENTS.md v1 table:
- Its `Phase` assignment should appear in ROADMAP.md as either Active or Completed
- If REQ-ID has no Phase, or Phase is "TBD" → orphan REQ — drift.

**Check D3: ROADMAP vs STATE alignment**

- ROADMAP.md `Active Milestone` name matches STATE.md `Current Milestone`
- If ROADMAP.md `Active Milestone` has Status = `complete` but STATE.md says `in_progress` → contradiction
- If ROADMAP.md `Active Milestone` doesn't appear at all in STATE.md → orphan milestone

**Check D4: Mode consistency**

- STATE.md `Approved Mode` should appear in latest `.planning/*-SUMMARY.md` (if any) — Mode change mid-project is suspicious
- If mode differs between STATE.md and most recent SUMMARY without an explicit re-gate event → drift

**Output**: list drift findings. **WARN** if recoverable (user can pick which to trust), **FAIL** if irreconcilable (need user decision before proceeding).

## Output Format

Always emit a structured report:

```
=== State Validation Report ===
Project: <PROJECT.md vision first line>
Branch: <git branch>
Checked: <YYYY-MM-DD HH:MM>

Category 1 — Existence: <PASS | FAIL>
  <details if FAIL>

Category 2 — Schema: <PASS | FAIL>
  <list errors per file if any>

Category 3 — Freshness: <PASS | WARN>
  <list warnings if any>

Category 4 — Drift: <PASS | WARN | FAIL>
  <list findings if any>

=== Verdict ===
<CLEAN | WARN | FAIL>

<if WARN or FAIL>
Suggested fixes:
  1. <specific actionable fix>
  2. <...>

Proceed? [accept_drift / fix_now / cancel]
</if>
```

## Verdict Semantics

| Verdict | Meaning | What to do |
|---|---|---|
| **CLEAN** | All 4 categories PASS | Continue with current workflow step |
| **WARN** | Schema + existence PASS but freshness or recoverable drift | Display report, ask user to accept drift or fix. Default: ask user. |
| **FAIL** | Missing files, schema errors, or irreconcilable drift | **BLOCK** — must fix before any other step proceeds |

## How to Fix Common Findings

### Missing required file
- Run `/init-project --force` (with backup) OR
- Manually create from `docs/claude/templates/`

### STATE.md last_updated stale after commit
- Determine the most recent step actually completed
- Update STATE.md: `Phase`, `Status`, `Next Action`, `Last updated`
- Commit if `commit_docs.state_files: true`

### REQ-ID schema error
- Rename to `^[A-Z]{2,8}-\d{2,4}$` pattern
- Update all references in PLAN/SUMMARY files

### ROADMAP/STATE milestone mismatch
- Pick the source of truth (usually STATE.md, since it's more frequently updated)
- Sync the other file to match
- Add Key Decision entry in STATE.md explaining the resolution

### Mode inconsistency
- Re-run Mode Selection Gate (STEP 3 of workflow)
- Update STATE.md `Approved Mode` to the gate output
- Document the mode change reason in STATE.md Key Decisions

## Cross-Platform Notes

This skill describes **logic**, not commands. The AI translates checks to available tools per harness:

- **Read files**: Use the harness's read tool (`Read`, `read_file`, etc.)
- **List directory**: `ls` (bash), `Get-ChildItem` (PowerShell), or harness equivalent
- **Get file mtime**: `stat -c %Y` (POSIX), `(Get-Item file).LastWriteTime` (PowerShell), `git log -1 --format=%ct -- file` (cross-platform)
- **Latest commit date**: `git log -1 --format=%ct` (universal)
- **Regex match**: AI evaluates in-context, no shell tool needed

No harness-specific code goes into this skill. Logic stays portable.

## Red Flags

**Never:**
- Proceed past STEP 0 without running this skill on resume
- Treat WARN as PASS — drift is a real signal, not noise
- Auto-fix state files without showing user the diff first
- Skip validation when "I'm sure state is fresh" — the rationalization itself is the bug

**Always:**
- Run before any step that reads STATE.md as input (i.e., almost every step)
- Show full report — never summarize away findings
- Get explicit user choice on `accept_drift | fix_now | cancel` when WARN
- Block on FAIL until resolved

## Integration

**Called by:**
- **STEP 0 (Bootstrap)** of `docs/claude/current-process-workflow.md` — MANDATORY on resume
- **STEP 7 (Execute)** — recommended pre-flight before dispatching tasks
- On-demand when user says "validate state" or "check state"

**Pairs with:**
- `skills/init-project/SKILL.md` — invoked if files missing
- `skills/update-config/SKILL.md` — invoked if config.json drift detected
- `docs/claude/state-files-guide.md` — source of truth for what each file should contain

**Inputs:**
- `STATE.md`, `PROJECT.md`, `REQUIREMENTS.md`, `ROADMAP.md` (project root)
- `.planning/*.md` (all artifacts)
- `.planning/config.json` (for granularity check)
- `git log` (for commit freshness)

**Outputs:**
- Structured report (above)
- Verdict: CLEAN | WARN | FAIL
- User decision: accept_drift | fix_now | cancel
