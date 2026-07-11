---
phase: process-2.0
plan: 03
type: execute
wave: 3
depends_on: ["01", "02"]
files_modified: [CLAUDE.md, docs/claude/templates/CLAUDE.md, docs/claude/current-process-workflow.md, docs/claude/templates/phase-VERIFICATION.md]
autonomous: false
requirements: [MEM-01, MEM-04, VER-01, VER-02, MEM-03]

must_haves:
  truths:
    - "With workflow.memory_recall on, STEP 0 recalls relevant lessons before the 'keep or edit config?' prompt"
    - "With workflow.sandbox_verify on, STEP 8 runs a bounded execute->observe->fix loop whose evidence lands in VERIFICATION, while UAT + Decision stay human"
    - "STEP 11 fires the MEM-03 flag-and-report pass and writes process-lessons to .claude/memory/ (distinct from the codebase-fact Knowledge Sync)"
    - "With every Process 2.0 flag off, the workflow behaves exactly as before"
  artifacts:
    - "CLAUDE.md"
    - "docs/claude/templates/CLAUDE.md"
    - "docs/claude/current-process-workflow.md"
    - "docs/claude/templates/phase-VERIFICATION.md"
  key_links:
    - "STEP 0 recall reads the .claude/memory/ store defined in memory-store-guide.md"
    - "STEP 11 process-lesson write-back targets .claude/memory/, NOT .claude/skills/<project>-* (that stays Knowledge Sync)"
    - "STEP 8 verify writes to the VERIFICATION Evidence column, separate from the human UAT Decision"
---

<!--
  Source: docs/specs/2026-07-11-process-2.0-memory-observability-design.md (D2, D3, D5).
  Wave 3: single task — CLAUDE.md + current-process-workflow.md are edited by STEP 0/8/11 concerns
  simultaneously, so one opus task owns them to avoid file-overlap conflicts and keep pillar-preserving
  wiring coherent. Depends on the mechanisms from plans 01 + 02.
-->

<wave number="3" description="Workflow integration — shared files, sequential, governance-sensitive">

  <task type="auto">
    <name>Wire STEP 0 recall, STEP 8 sandbox-verify, STEP 11 MEM-03 + memory write-back into the workflow docs</name>
    <model>opus</model>
    <read_first>
      docs/specs/2026-07-11-process-2.0-memory-observability-design.md
      CLAUDE.md
      docs/claude/templates/CLAUDE.md
      docs/claude/current-process-workflow.md
      docs/claude/templates/phase-VERIFICATION.md
      docs/claude/memory-store-guide.md
      skills/memory-maintenance/SKILL.md
    </read_first>
    <files>
      <modify>CLAUDE.md</modify>
      <modify>docs/claude/templates/CLAUDE.md</modify>
      <modify>docs/claude/current-process-workflow.md</modify>
      <modify>docs/claude/templates/phase-VERIFICATION.md</modify>
    </files>
    <action>
      REQ MEM-01, MEM-04, VER-01, VER-02, MEM-03 (inline). Wire the capabilities into the 11-step workflow,
      every one CONFIG-GATED and preserving the 3 pillars (never auto-advance · REQ-ID traceability ·
      AI does not self-declare done). Mirror each STEP edit in BOTH CLAUDE.md and docs/claude/templates/CLAUDE.md,
      and in docs/claude/current-process-workflow.md.

      - STEP 0 (MEM-01/04), gated by `workflow.memory_recall`: after reading STATE.md + config.json and BEFORE the
        "keep or edit?" prompt, run a recall step that greps the `.claude/memory/` store (per memory-store-guide.md)
        for top-K relevant lessons and surfaces them. This is an agent step (SessionStart hook does not fire on
        mid-session resume). No-op when the flag is off or the store is absent.
      - STEP 8 (VER-01/02), gated by `workflow.sandbox_verify`: for a task that produced runnable code, add a
        BOUNDED execute->observe->fix loop: hard cap of 3 fix iterations; each fix traces to the task's REQ-ID and
        stays within the task's planned files (Surgical Changes); NO permission escalation (same Bash allowlist);
        evidence-only — write run evidence into the phase-VERIFICATION.md Evidence column, in a section separate
        from UAT. STEP 8a UAT and the human Decision (PASS/PARTIAL/FAIL) remain unskippable. Verify NEVER auto-passes STEP 8.
      - STEP 11 (MEM-03 inline + memory write-back): add a flag-and-report MEM-03 sub-step (invokes memory-maintenance,
        flags only, human approves). Add process-lesson write-back to `.claude/memory/` (English-normalized), placed
        alongside — but DISTINCT from — the existing Knowledge Sync (codebase facts -> .claude/skills/<project>-*).
        State the boundary explicitly so a lesson lands in exactly one home.
      - phase-VERIFICATION.md: add a one-line note that sandbox-verify run evidence goes in the Evidence column,
        kept separate from the human UAT approval.
      - Register docs/claude/memory-store-guide.md in the "Primary references" list in BOTH CLAUDE.md and
        docs/claude/templates/CLAUDE.md (discoverability; per Plan Checker suggestion D).

      Keep edits additive and surgical; do not restructure unrelated workflow text. Do not alter any gate to auto-advance.
    </action>
    <verify>
      <automated>grep -li "sandbox.\?verify\|Evidence" docs/claude/templates/phase-VERIFICATION.md && grep -c "memory_recall\|sandbox_verify\|memory-clean\|\.claude/memory" CLAUDE.md docs/claude/templates/CLAUDE.md docs/claude/current-process-workflow.md</automated>
    </verify>
    <done>
      STEP 0 recall (memory_recall-gated), STEP 8 bounded evidence-only verify (sandbox_verify-gated, cap 3, UAT+Decision
      human), and STEP 11 MEM-03 + .claude/memory write-back (distinct from Knowledge Sync) are present in CLAUDE.md,
      templates/CLAUDE.md, and current-process-workflow.md; phase-VERIFICATION.md notes the Evidence/UAT separation;
      grep confirms the gating keywords; with all flags off the described behavior is unchanged.
    </done>
  </task>

</wave>
