---
name: memory-maintenance
description: Use when the process-memory store (.claude/memory/) needs cleaning — duplicate, stale, superseded, or contradictory lessons are cluttering recall — or at STEP 11 Ship to flag removal candidates. Triggered by /memory-clean, "clean memory", "prune process-memory", "memory is noisy / has dupes", or the inline STEP-11 flag-and-report pass.
---

# Memory Maintenance

Keeps the **process-memory store** (`.claude/memory/`) trustworthy by scanning it for entries that no longer earn their place and **flagging them for human approval**. Recall (MEM-04) is only as good as the store is clean; this skill is how the store stays clean without ever letting the AI silently rewrite the team's memory.

> **This skill flags, it does not delete.** It never removes, edits, or merges a lesson on its own. Every removal is proposed to the human, who approves each one individually. This is pillar #1 (never auto-advance) applied to memory — the AI does not get to quietly forget things on the team's behalf.

Store schema, location, and boundary rules: `docs/claude/memory-store-guide.md` (the contract this skill operates on). Design source: `docs/specs/2026-07-11-process-2.0-memory-observability-design.md` (D3).

## When to use

- The process-memory store has grown noisy — recall is surfacing duplicates or lessons that are plainly out of date.
- A lesson has been **superseded** (a newer lesson states the opposite, or the config/model default it warned about has changed).
- Two lessons **contradict** each other and recall can't be trusted to pick the right one.
- User runs `/memory-clean`.
- **STEP 11 Ship** runs its inline flag-and-report pass (see Entry points).

## When NOT to use

- **The store is absent or tiny.** No `.claude/memory/` → no-op (say so and stop). A handful of clean lessons needs no maintenance.
- **You want to ADD a lesson.** That's the STEP 11 capture write-back (feeds MEM-01/04 recall), not this skill. This skill only prunes.
- **You want to clean the knowledge library** (`.claude/skills/<project>-*`). Out of bounds — that store is curated and human-gated by `mining-project-knowledge`; this skill writes nothing there. See the boundary rule in `memory-store-guide.md`.
- **You want to change what recall returns without removing anything.** Tune the recall query (MEM-04), don't delete lessons.

## Ground rules (non-negotiable)

1. **Never auto-delete.** The skill produces a *proposal*; the human approves each removal. No batch "yes to all" is offered by the AI — it presents candidates and waits.
2. **Write fence.** Operate ONLY inside `.claude/memory/`. Never touch `.claude/skills/<project>-*` (the knowledge library) or anything outside the store.
3. **No-op when absent.** If `.claude/memory/` does not exist, do nothing and report it — this matches the conditional-hook convention.
4. **Evidence per flag.** Every flagged candidate cites *why* (which other lesson it duplicates/contradicts, or what makes it stale) so the human can decide in one read.
5. **Update over delete, when it applies.** Reuse the existing memory guideline: prefer proposing an *update/merge* of an outdated note over deleting it, when the lesson still holds in part. Deletion is for lessons that are wrong or fully superseded.

## Flag categories

The scan sorts candidates into four buckets. Each candidate is presented with its file, its bucket, and the evidence.

| Category | What it means | How it's detected |
|---|---|---|
| **Duplicate** | Two+ lessons say the same thing. | High keyword/`tags` overlap + same `type`; near-identical bodies. |
| **Stale** | The condition the lesson warned about no longer exists. | Old `date` **and** the referenced config/model-default/workflow behavior has since changed (verify against current config/docs before flagging). |
| **Superseded** | A newer lesson replaces it. | A later-dated lesson on the same `tags`/`type` states a different or opposite conclusion. |
| **Contradictory** | Two lessons give conflicting guidance and both are live. | Same `tags`/`type`, incompatible conclusions, neither clearly newer — flag the *pair* for the human to reconcile. |

Staleness is **evidence-based, not age-based** — an old lesson that is still true is not stale. Never flag on `date` alone.

## How it runs

1. **Locate the store.** Confirm `.claude/memory/` exists. Absent → report no-op and stop.
2. **Load the index + scan lessons.** Read `MEMORY.md` and the lesson files' frontmatter (`scope`, `type`, `tags`, `date`) and bodies. For a large store, scan in `tags`/`type` clusters rather than loading everything at once.
3. **Detect candidates** into the four buckets above, each with cited evidence. Verify staleness/supersession against the *current* config and docs — do not assume.
4. **Present the report** grouped by category (see Report format). Nothing is changed yet.
5. **Human approves per candidate.** For each one the human keeps/removes/edits. Only on explicit approval does the skill act.
6. **Apply approved removals** inside `.claude/memory/` only — delete the approved lesson files, or apply the approved update/merge, and **update `MEMORY.md`** to drop the removed index lines. Untouched candidates stay exactly as they were.
7. **Report what changed.** List removals/edits applied and what was kept. The controller commits (this skill does not `git commit`).

## Entry points

This skill is reachable **two** ways (both flag-only — neither deletes without approval):

### (a) On-demand — `/memory-clean`

The user runs the `/memory-clean` slash command whenever they want a cleanup pass. Runs the full flow above interactively: scan → report → approve → apply. `commands/memory-clean.md` invokes this skill.

### (b) Inline at STEP 11 Ship — flag-and-report pass

At **STEP 11 (Ship)**, a lightweight flag-and-report pass fires: it runs the **scan + report** (steps 1–4) and surfaces any removal candidates for the user to approve as part of Ship — so memory gets tended on the same cadence lessons are captured (feeds MEM-01/04 recall), without a separate chore. It **never auto-deletes**; if the user doesn't approve during Ship, the candidates simply carry forward to the next pass.

> The STEP-11 *wiring* (the call site in the workflow/CLAUDE.md STEP 11) is added separately (plan 03). This skill only *provides* the behavior the STEP-11 pass invokes — a scan-and-report that returns flagged candidates for approval.

## Report format

Group candidates by category; one row per candidate (or per pair for contradictions):

```
## Memory maintenance — <N> candidates flagged (nothing removed yet)

### Duplicate (2)
- haiku-underplans-multifile-waves.md  ⇄  haiku-multifile-escalation.md
  Evidence: same type=escalation, tags overlap [haiku, multi-file, wave]; near-identical body. Propose: merge into the earlier, drop the later.

### Stale (1)
- config-commit-atomic-batching.md
  Evidence: date 2026-05; warns about a default that changed in config-schema.md on 2026-07. Propose: remove (or update the note).

### Superseded (0)

### Contradictory (1)
- plan-check-loop.md  ⇄  plan-check-needs-map.md
  Evidence: conflicting guidance on the same tags [plan-check]; neither clearly newer. Needs human reconciliation.

Approve removals individually — reply keep / remove / edit per candidate.
```

## Provenance and maintenance

- Created 2026-07-11 from `docs/specs/2026-07-11-process-2.0-memory-observability-design.md` (D3), REQ MEM-03.
- Operates on the store defined in `docs/claude/memory-store-guide.md` — if that schema changes (frontmatter fields, location), update the scan logic here.
- Re-verify the never-auto-delete rule is stated: `grep -ci "flags, it does not delete\|never auto-delete\|human approves" skills/memory-maintenance/SKILL.md`.
- STEP-11 wiring lives in the workflow docs (plan 03) — if the call site moves, update Entry point (b) here to match.
