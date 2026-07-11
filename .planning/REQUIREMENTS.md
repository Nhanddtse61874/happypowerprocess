# Requirements — Process 2.0: Memory Recall, Observability & Sandbox Verification

**Source:** Brainstorm output 2026-07-11 (plugin self-improvement)
**Phase traceability:** Step 2 (Brainstorm) → Step 5 (Spec)
**Branch:** `feature/docs-consolidation-knowledge-layer-2026-07-08`

## Context

Process 2.0 absorbs the *strengths* of three runtime agent stacks into the existing
11-step workflow **without adopting them as external runtimes**:

- **Mem0** → semantic memory recall + at-scale indexing (STEP 0, cross-cutting)
- **DeerFlow 2.0** → sandbox self-verify loop + dynamic sub-agents (STEP 6–8)
- **MS Agent Framework 1.0** → telemetry/observability + checkpointing (STEP 7–11)

It **builds on**, and does not replace, the existing knowledge layer
(`mining-project-knowledge`, `docs/specs/2026-07-07-knowledge-mining-pilot-protocol.md`).
The knowledge library is a *curated, CSO-triggered, per-project skill set* ("what the
codebase IS"); Process 2.0 memory is *auto-captured lessons with on-demand recall* — a
different layer that feeds STEP 0/4 and is fed by STEP 11 Knowledge Sync.

**Non-negotiable — the 3 pillars are preserved by every requirement below:**
1. Never auto-advance (human gate after each step).
2. REQ-ID 100% traceability.
3. AI does not self-declare done (sandbox verify produces *evidence for* UAT, never *replaces* it).

## REQ-ID Format

`[CATEGORY]-[NUMBER]`. Categories: `MEM` (memory), `OBS` (observability),
`VER` (verification), `CFG` (config gating), `EXE` (dynamic execution).
Each requirement is specific & testable, user-centric (the "user" is a developer running
the workflow), atomic, and independent.

## v1 Requirements (Ship in initial release)

| REQ-ID | Requirement | Phase |
|---|---|---|
| MEM-01 | At STEP 0 Resume, a developer sees relevant prior lessons/decisions auto-recalled from accumulated memory before the "keep or edit config?" prompt — without manually searching memory files. | Phase 1 |
| MEM-02 | A developer's memory entries are scoped as global vs per-project, so per-project entries never bloat the global index loaded every session. | Phase 1 |
| MEM-03 | A developer can run a memory maintenance pass that flags duplicate / stale / superseded entries for removal, keeping the recalled index lean. | Phase 1 |
| MEM-04 | Memory recall retrieves only the top-relevant entries on demand (search-then-load) instead of loading the whole index every session, so per-session context cost stays flat as memory grows. | Phase 1 |
| OBS-01 | After a wave completes, a developer sees per-task telemetry (tokens, wall-clock, model used, escalations) captured to disk. | Phase 1 |
| OBS-02 | At STEP 11, the phase SUMMARY includes an aggregated telemetry table built from the captured per-task metrics. | Phase 1 |
| CFG-01 | A developer can enable/disable each Process 2.0 capability independently via a config flag, defaulting to conservative/off. | Phase 1 |
| VER-01 | At STEP 8, for a task producing runnable code, an agent runs a sandbox self-verify loop (execute → observe → fix) and attaches run evidence to the VERIFICATION artifact before UAT. | Phase 2 |
| VER-02 | A developer sees sandbox-verify evidence as input to UAT/QA, while those gates still require explicit approval — verify never auto-passes a step. | Phase 2 |

## v2 Requirements (Deferred — table stakes users expect)

| REQ-ID | Requirement | Reason deferred |
|---|---|---|
| MEM-05 | When active memory count crosses a configurable threshold (default 120), a hook bootstraps a semantic vector index once, then syncs it incrementally; files stay the git source of truth, index is a rebuildable cache. | Tier-1 external deps (vector store + embeddings); only needed beyond a few hundred memories — recall-on-demand (MEM-04) covers earlier scale. |
| EXE-01 | Within a wave, the controller can spawn a sub-agent on demand for an ambiguous/blocked sub-task, not only the statically planned agents. | Governance risk — must be designed so it cannot bypass gates; prove the safer v1 items first. |
| OBS-03 | Telemetry rolls up across phases/projects into a trend view. | Only after per-task telemetry (OBS-01/02) is proven in the field. |

## Out of Scope (Explicit exclusions)

| Item | Reason |
|---|---|
| Running MS Agent Framework / DeerFlow / Mem0 as external runtimes replacing Claude Code | Loses the governance layer; the workflow stays in Claude Code and only absorbs ideas. |
| Replacing the existing knowledge library (`mining-project-knowledge`) | Process 2.0 complements it (different layer); the knowledge library stays authoritative for "what the codebase IS". |
| Auto-advancing any step or auto-passing any gate | Violates pillar #1 and #3. |

## Assumptions

- Memory files are committed to git as the portable source of truth; any vector index (MEM-05) is a rebuildable cache, so a new machine re-derives it rather than losing knowledge.
- v1 (MEM-01..04, OBS-01/02, VER-01/02) is implemented with existing Claude Code primitives — memory files, hooks in `settings.json`, the Bash sandbox — with **no new runtime dependency**. Only MEM-05 (v2) introduces external deps.
- Every capability is config-gated (CFG-01) and defaults conservative, so Process 2.0 never forces overhead onto a task that doesn't want it.
- Work continues on branch `feature/docs-consolidation-knowledge-layer-2026-07-08`.
- Phase mapping (to be formalized in ROADMAP.md at STEP 6 / planning): **Phase 1** = Memory recall + scaling + Observability; **Phase 2** = Sandbox self-verification.

## Last updated

2026-07-11
