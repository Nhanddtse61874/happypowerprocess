# Process 2.0 — Phase Summary

**Phase:** process-2.0 (Memory Recall, Observability, Sandbox Verification) — plugin self-improvement
**Completed:** 2026-07-11
**Mode:** A (Solo) · model_profile: quality · all tasks opus
**Released as:** v6.0.0
**Branch:** `feature/docs-consolidation-knowledge-layer-2026-07-08`

## What Was Built

Three **opt-in, default-off** capabilities absorbed into the 11-step workflow from the strengths of Mem0 / DeerFlow 2.0 / MS Agent Framework — built entirely on existing Claude Code primitives, **zero new runtime dependency** in v1.

- **Memory (MEM-01..04)** — process-memory store at `<project>/.claude/memory/` (git-committed, portable); STEP-0 recall-on-demand (grep top-K, agent step); scope tags (global/project); `/memory-clean` + STEP-11 flag-and-report maintenance (flags only, human approves); English-normalized write-back. New: `docs/claude/memory-store-guide.md`, `skills/memory-maintenance/SKILL.md`, `commands/memory-clean.md`.
- **Observability (OBS-01/02)** — per-task telemetry (token optional) appended to `.planning/{phase}-telemetry.jsonl` from the STEP-7 controller loop (not a hook); aggregated into the STEP-11 SUMMARY table + `phase-summary-v1` contract.
- **Verification (VER-01/02)** — STEP-8 bounded (cap 3) evidence-only execute→observe→fix; scope-fenced to REQ-ID, no permission escalation; UAT + human Decision preserved.
- **Config (CFG-01)** — `workflow.memory_recall / telemetry / sandbox_verify`, default false, in both `config-schema.md` + `templates/config.json`.

## Key Decisions

- **Two memory stores, hard boundary:** process-memory (`how we worked`) vs the existing knowledge library (`what the code is`, `.claude/skills/<project>-*`) — one home per fact; process-memory never writes the knowledge namespace.
- **`.claude/memory/` (not `.planning/`)** so `commit_docs.planning_artifacts:false` can't silently un-commit it → portable across machines.
- **Telemetry from the controller, not a hook** — tokens/model/escalation are controller-side, and Windows hooks silently no-op without Git Bash.
- **MEM-05 semantic vector index (@120 threshold) deferred to v2** — lexical recall + scoping + maintenance covers early scale with zero deps.
- **All capabilities default-off** — workflow byte-for-byte unchanged until a project opts in.

## Commits (13)

Planning (`d423017`) → CFG-01 (`c7e77ca`) → MEM-02 guide (`7318102`) → MEM-03 (`abd77fb`) → OBS-01 (`273cadc`) → OBS-02 (`98563d9`) → workflow wiring (`fdc9514`) → QA telemetry-enum fix (`9517996`, `f8ae982`) → audit gap-closure (`d19635e`) → release docs (`8d6b240`) → version bump v6.0.0 (`63b63d3`).

## What Worked

- **Plan Checker** caught the Plan-03 verify-coverage gap before execution (Defect A).
- **QA Gate** caught the telemetry `result` enum mismatch (writer vs contract).
- **Pre-release audit (3 agents)** caught the two issues that would actually have shipped broken: the `MEM-01`/`MEM-04` REQ-ID contradiction (recall vs write-back) across 6 files, and a fully-stale README + RELEASE-NOTES. Auditing *before* the version bump paid off.
- Wave structure (mechanisms parallel in W1/W2, shared-file wiring serialized in W3) held with zero conflicts.

## What Didn't Work / Lessons

- **REQ-ID drift during wiring (Wave 3):** the write-back got mis-tagged `MEM-01` and `memory-store-guide` swapped MEM-01/MEM-04 — pillar-#2 traceability slipped and was only caught at the pre-release audit. Lesson: cross-file REQ-ID consistency needs an explicit check when the same capability spans STEP 0/8/11 mirrors.
- **Release-facing docs (README/RELEASE-NOTES) were not in the plan** — they belong in the STEP-6 plan for any versioned release, not discovered at ship time.
- **QA S-2/S-3 "logged" then re-surfaced** in the audit as real (MEM-01 mis-tag, sandbox_verify no-op parity) — logging a suggestion isn't closing it; both were fixed in `d19635e`.

## Residual Risks

- **Documented + wired, not runtime-exercised** — every capability is default-off; first real activation on a project is the field test.
- **Lexical recall precision** (grep) degrades before file *count* does; MEM-05 (v2) is the semantic upgrade. MEM-03 pruning keeps recall trustworthy meanwhile.
- **Telemetry `tokens`** depends on the harness exposing a count; omitted (never fabricated) when unavailable.

## Telemetry

N/A — `workflow.telemetry` was off during this phase (dogfooding deferred; the capability ships default-off).

## Next Milestone Input (v2)

- **MEM-05** — semantic vector index auto-built at the configurable threshold (default 120) via a hook, incremental sync, files as source-of-truth.
- **EXE-01** — dynamic sub-agent spawning (same "cannot bypass gate" bar as VER-01).
- **OBS-03** — cross-phase/project telemetry rollup.
- **Field test** — enable the three flags on a real project and capture field notes to harden v1.
