# Design: Workflow Refactor — Brainstorm-First with Mode Selection Gate

**Date:** 2026-04-06
**Branch:** feature/workflow-brainstorm-first-2026-04-06
**Status:** Approved

---

## Problem Statement

The current workflow selects runtime mode (Mode A or Mode B) before brainstorming. This creates a fundamental sequencing problem: mode selection requires understanding task complexity, but complexity is only understood after brainstorming. Additionally, in Mode B, Spec and Implementation Plan were written solo despite team agents being available and appropriate for that phase.

---

## Goals

1. Move brainstorming before mode selection in all non-Fast-Lane paths.
2. Introduce a Mode Selection Gate as a required output of brainstorming.
3. Create a dedicated `mode-selection-criteria.md` file as the authoritative reference for evaluating mode complexity.
4. Delegate Spec and Implementation Plan writing in Mode B to team agents (Discovery Lead + Architecture Lead + Implementation Lead), using brainstorm output as input artifact.
5. Enforce that user-approved mode is the source of truth for all downstream phases.
6. Update all affected files: CLAUDE.md, docs/claude/, skills/, agents/.

---

## New Workflow Order

### Non-Fast-Lane Path

```
Session Bootstrap (using-superpowers skill)
    ↓
Brainstorm (brainstorming skill)
  → problem · scope · constraints · 2-3 approaches · approved design direction
    ↓
Mode Selection Gate
  → read mode-selection-criteria.md
  → score each criterion
  → present: reason + suggested mode (A or B)
  → USER APPROVES or OVERRIDES
    ↓
[Mode A path] or [Mode B path] — follows user-approved mode exactly
```

### Fast Lane Path (hotfix/small task)

```
Session Bootstrap
    ↓
Fast Lane Eligibility Check (fast-lane-assessment-v1)
  → eligible: skip brainstorm
    ↓
Mode Selection Gate (abbreviated)
  → USER APPROVES mode
    ↓
[Mode A path] or [Mode B path]
```

---

## Mode Selection Gate — Specification

### Trigger
Runs immediately after brainstorm output is complete (or after Fast Lane assessment for Fast Lane tasks).

### Input
- Brainstorm output artifact (problem, scope, constraints, approved direction)
- `docs/claude/mode-selection-criteria.md` (read at evaluation time)

### Evaluation
Claude scores each criterion defined in `mode-selection-criteria.md` and produces:

```
Complexity evaluation:
  - Domain count: [1 / 2 / 3+]
  - Risk level: [low / medium / high]
  - QA/DevOps gate needed: [yes / no]
  - Cross-team coordination: [yes / no]
  - Output format required: [informal / machine-readable]

Suggested mode: A or B
Reason: [specific, based on scored criteria]
```

### User Authority
User approves or overrides the suggested mode. Once approved:
- User-approved mode is the source of truth for all downstream phases.
- Claude does not re-evaluate or adjust mode after user approval.
- If user chooses Mode A despite Mode B suggestion, all phases (spec, plan, implementation, QA, finish) run in Mode A.

---

## Mode A Workflow (unchanged structure, updated entry)

1. Session Bootstrap
2. Brainstorm → Mode Selection Gate → User approves Mode A
3. Design Spec — `brainstorming skill` → `docs/specs/YYYY-MM-DD-design.md` (user-approved)
4. Implementation Plan — `writing-plans skill`
5. Implementation — `subagent-driven-development` or `executing-plans` + mandatory stack skill
6. TDD — `test-driven-development skill` RED→GREEN→REFACTOR
7. Code Review — `requesting-code-review skill`
8. Finish Branch — `finishing-a-development-branch skill`

---

## Mode B Workflow — Spec & Plan via Team Agents (key change)

1. Session Bootstrap
2. Brainstorm → Mode Selection Gate → User approves Mode B
3. Orchestration Intake — `team-orchestrator`
   - Input: brainstorm output as context artifact
   - Output: `orchestrator-status-v1`
4. Dispatch — `master-dispatcher`
   - Routes spec task → Discovery Lead + Architecture Lead
   - Routes plan task → Implementation Lead
   - Output: Dispatcher JSON Contract
5. Discovery Phase — `phase-discovery-lead`
   - Input: brainstorm output (no re-brainstorming)
   - Task: formalize requirements spec from brainstorm output
   - Output: `phase-lead-report-v1`
6. Architecture Phase — `phase-architecture-lead`
   - Input: brainstorm output + discovery report
   - Task: formalize technical spec (contracts, interfaces, trade-offs)
   - Output: `phase-lead-report-v1` or `specialist-report-v1`
7. Spec Review
   - Orchestrator internal review gate
   - Then: present spec to user → User approves or requests changes
   - Written to: `docs/specs/YYYY-MM-DD-design.md`
8. Implementation Plan — `phase-implementation-lead`
   - Input: approved spec
   - Task: write ordered, testable implementation plan
   - Output: `phase-lead-report-v1`
9. Plan Review
   - Orchestrator internal review gate
   - Then: present plan to user → User approves or requests changes
10. Implementation — dispatcher-selected stack implementer + mandatory stack skill
    - Output: `implementer-delivery-v1`
11. QA Gate — `phase-qa-gate` + `qa-code-reviewer` + `requesting-code-review skill`
    - Output: `qa-review-v1`
    - Block → Fix → Re-review loop
12. Release & Ops — `phase-release-devops-lead`
    - Output: `devops-release-v1`
13. Final Handoff — `team-orchestrator`
    - Output: `orchestrator-status-v1`
14. Escalation (if risk/conflict detected) — 2-3 options → User decides

---

## New File: mode-selection-criteria.md

Location: `docs/claude/mode-selection-criteria.md`

Content:
- Scoring matrix with 5 criteria (domain count, risk level, QA/DevOps need, cross-team coordination, output format)
- Threshold rules: when each criterion pushes toward Mode A vs Mode B
- Hard exclusions: cases that always require Mode B regardless of other scores
- Borderline examples with worked scoring
- Override rule: user decision always wins

---

## Files Affected

| File | Change Type | Summary |
|---|---|---|
| `docs/claude/mode-selection-criteria.md` | CREATE | New scoring matrix file |
| `CLAUDE.md` | UPDATE | Execution rule: brainstorm-first + mode gate |
| `docs/claude/current-process-workflow.md` | UPDATE | New workflow order for both modes |
| `docs/claude/runtime-modes.md` | UPDATE | Remove inline selection rules, point to criteria file |
| `docs/claude/master-dispatcher-prompt.md` | UPDATE | Accept brainstorm output as input for spec/plan tasks |
| `docs/claude/full-ai-team-setup.md` | UPDATE | Execution pattern and phase ownership |
| `skills/brainstorming/SKILL.md` | UPDATE | Add Mode Selection Gate as required output |
| `skills/writing-plans/SKILL.md` | UPDATE | Add Mode B path: input = brainstorm output + approved spec |
| `agents/team-orchestrator.md` | UPDATE | Accept brainstorm context, own spec/plan review gate |
| `agents/master-dispatcher.md` | UPDATE | Route spec/plan to discovery/architecture/implementation leads |
| `agents/phase-discovery-lead.md` | UPDATE | Formalize spec from brainstorm output (no re-brainstorm) |
| `agents/phase-architecture-lead.md` | UPDATE | Formalize technical spec from brainstorm output |
| `agents/phase-implementation-lead.md` | UPDATE | Write plan from approved spec |
| `docs/claude/workflow-diagram.md` | DONE | Already updated in previous session |

---

## Key Rules Summary

1. **Brainstorm before mode selection** — no exception except Fast Lane.
2. **Mode Selection Gate is mandatory** — cannot skip after brainstorm.
3. **User approves mode** — Claude suggests, user decides.
4. **User-approved mode = source of truth** — all phases downstream follow it without exception.
5. **Brainstorm output is the input artifact** — Discovery Lead and Architecture Lead consume it, do not re-brainstorm.
6. **Double review gate in Mode B spec/plan** — orchestrator internal review + user final approve.
7. **Fast Lane still skips brainstorm** — but still runs abbreviated Mode Selection Gate.
