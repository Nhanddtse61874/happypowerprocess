---
phase: process-2.0
plan: 02
type: execute
wave: 2
depends_on: ["01"]
files_modified: [commands/memory-clean.md, skills/memory-maintenance/SKILL.md, skills/subagent-driven-development/SKILL.md, docs/claude/templates/phase-SUMMARY.md, docs/claude/agent-output-templates.md, skills/finishing-a-development-branch/SKILL.md]
autonomous: false
requirements: [MEM-03, OBS-01, OBS-02]

must_haves:
  truths:
    - "A developer can run /memory-clean to flag duplicate/stale memory entries for approval (never auto-deleted)"
    - "When workflow.telemetry is on, per-task metrics are appended to .planning/{phase}-telemetry.jsonl during STEP 7"
    - "When workflow.telemetry is on, the STEP 11 SUMMARY renders an aggregated telemetry table"
  artifacts:
    - "commands/memory-clean.md"
    - "skills/memory-maintenance/SKILL.md"
    - "skills/subagent-driven-development/SKILL.md"
    - "docs/claude/templates/phase-SUMMARY.md"
  key_links:
    - "finishing-a-development-branch reads .planning/{phase}-telemetry.jsonl written by subagent-driven-development"
    - "memory-maintenance flags candidates but a human approves every deletion"
---

<!-- Source: docs/specs/2026-07-11-process-2.0-memory-observability-design.md (D3, D4). Wave 2: disjoint files, parallel-safe. -->

<wave number="2" description="Mechanisms — disjoint files, parallel-safe">

  <task type="auto">
    <name>MEM-03 maintenance: /memory-clean command + memory-maintenance skill</name>
    <model>opus</model>
    <read_first>
      docs/claude/memory-store-guide.md
      docs/specs/2026-07-11-process-2.0-memory-observability-design.md
      commands/mine-knowledge.md
      skills/mining-project-knowledge/SKILL.md
    </read_first>
    <files>
      <create>commands/memory-clean.md</create>
      <create>skills/memory-maintenance/SKILL.md</create>
    </files>
    <action>
      REQ MEM-03 + design D3. Create a slash-command + skill pair (English):
      - skills/memory-maintenance/SKILL.md: scans the process-memory store (per memory-store-guide.md),
        FLAGS duplicate / stale / superseded / contradictory entries and presents them for human approval.
        It MUST NOT auto-delete (pillar #1) — the human approves every removal. Document BOTH entry points:
        (a) on-demand via /memory-clean, (b) an inline flag-and-report pass fired at STEP 11 (the wiring for
        the STEP 11 call itself lives in plan 03). Frontmatter `description` = triggers only ("Use when...").
      - commands/memory-clean.md: slash-command entry point that invokes the skill (mirror commands/mine-knowledge.md pattern).
      Do NOT wire STEP 11 here (plan 03 owns the workflow docs). Do NOT implement recall (plan 03/STEP 0).
    </action>
    <verify>
      <automated>test -f commands/memory-clean.md && grep -i "does not\|never\|human approv\|flag" skills/memory-maintenance/SKILL.md | head -1</automated>
    </verify>
    <done>
      Both files exist; skill flags candidates and states it never auto-deletes (human approves); command invokes the skill;
      skill documents both /memory-clean and the STEP-11 flag-and-report entry points.
    </done>
  </task>

  <task type="auto">
    <name>OBS-01 telemetry capture in the STEP 7 controller loop</name>
    <model>opus</model>
    <read_first>
      skills/subagent-driven-development/SKILL.md
      docs/specs/2026-07-11-process-2.0-memory-observability-design.md
      docs/claude/config-schema.md
    </read_first>
    <files>
      <modify>skills/subagent-driven-development/SKILL.md</modify>
    </files>
    <action>
      REQ OBS-01 + design D4. At the existing per-task controller step (the "update STATE.md after each task"
      block), add a telemetry write GATED by `workflow.telemetry` (no-op when off):
      append ONE line to `.planning/{phase}-telemetry.jsonl` per task — append-only JSONL (crash-tolerant;
      a partial line never corrupts prior data). Fields: `task_id`, `req_ids`, `model`, `wall_clock`,
      `escalation` (e.g. "haiku->sonnet"), `result`; include `tokens` ONLY when the harness exposes it —
      never fabricate a token count. Keep it a documented controller-loop behavior, not a hook.
    </action>
    <verify>
      <automated>grep -c "telemetry.jsonl\|workflow.telemetry\|append" skills/subagent-driven-development/SKILL.md</automated>
    </verify>
    <done>
      Skill documents a flag-gated, append-only per-task telemetry write to .planning/{phase}-telemetry.jsonl
      with tokens optional; no-op when workflow.telemetry is false.
    </done>
  </task>

  <task type="auto">
    <name>OBS-02 telemetry aggregation into the phase SUMMARY</name>
    <model>opus</model>
    <read_first>
      docs/claude/templates/phase-SUMMARY.md
      docs/claude/agent-output-templates.md
      skills/finishing-a-development-branch/SKILL.md
      docs/specs/2026-07-11-process-2.0-memory-observability-design.md
    </read_first>
    <files>
      <modify>docs/claude/templates/phase-SUMMARY.md</modify>
      <modify>docs/claude/agent-output-templates.md</modify>
      <modify>skills/finishing-a-development-branch/SKILL.md</modify>
    </files>
    <action>
      REQ OBS-02 + design D4. Wire aggregation, GATED by `workflow.telemetry`:
      - phase-SUMMARY.md: add a "Telemetry" section (a table: task | model | wall_clock | escalation | tokens?),
        rendered only when telemetry data exists; the `tokens` column shows only when present.
      - agent-output-templates.md: add a `telemetry` key to the `phase-summary-v1` contract.
      - finishing-a-development-branch/SKILL.md: at SUMMARY generation, read `.planning/{phase}-telemetry.jsonl`
        and render the table; no-op when the flag is off or the file is absent.
      Do NOT change any non-telemetry section of the SUMMARY template or contract.
    </action>
    <verify>
      <automated>grep -il "telemetry" docs/claude/templates/phase-SUMMARY.md docs/claude/agent-output-templates.md skills/finishing-a-development-branch/SKILL.md</automated>
    </verify>
    <done>
      SUMMARY template has a Telemetry section; phase-summary-v1 contract has a telemetry key;
      finishing skill reads the jsonl and renders the table gated by workflow.telemetry.
    </done>
  </task>

</wave>
