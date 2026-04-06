# Workflow Refactor: Brainstorm-First with Mode Selection Gate — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor the entire Superpowers Personalized Plugin workflow so that Brainstorming always runs before Mode Selection, Mode Selection uses a dedicated scoring file, and Mode B Spec/Plan is delegated to team agents.

**Architecture:** All changes are to markdown documentation and skill/agent definition files — no code. Each task targets one file, makes a focused edit, verifies the result, and commits. Tasks are ordered by dependency: shared reference files first, consumer files after.

**Tech Stack:** Markdown, Git

---

## File Map

| File | Action | Purpose |
|---|---|---|
| `docs/claude/mode-selection-criteria.md` | CREATE | New authoritative scoring matrix for mode selection |
| `CLAUDE.md` | MODIFY | Update execution rule to brainstorm-first |
| `docs/claude/current-process-workflow.md` | MODIFY | New workflow order: brainstorm → mode gate → mode path |
| `docs/claude/runtime-modes.md` | MODIFY | Remove inline selection rules, point to criteria file |
| `docs/claude/master-dispatcher-prompt.md` | MODIFY | Accept brainstorm output as input for spec/plan |
| `docs/claude/full-ai-team-setup.md` | MODIFY | Update execution pattern and phase ownership |
| `skills/brainstorming/SKILL.md` | MODIFY | Add Mode Selection Gate as required checklist item |
| `skills/writing-plans/SKILL.md` | MODIFY | Add Mode B path: input = brainstorm output + approved spec |
| `agents/team-orchestrator.md` | MODIFY | Accept brainstorm context, own spec/plan review gate |
| `agents/master-dispatcher.md` | MODIFY | Route spec/plan tasks to correct phase leads |
| `agents/phase-discovery-lead.md` | MODIFY | Formalize spec from brainstorm output, no re-brainstorm |
| `agents/phase-architecture-lead.md` | MODIFY | Formalize technical spec from brainstorm output |
| `agents/phase-implementation-lead.md` | MODIFY | Write plan from approved spec |

---

### Task 1: Create docs/claude/mode-selection-criteria.md

**Files:**
- Create: `docs/claude/mode-selection-criteria.md`

- [ ] **Step 1: Create the file with full scoring matrix content**

```markdown
# Mode Selection Criteria

This file is the authoritative reference for evaluating task complexity and selecting a runtime mode.
Claude MUST read this file during the Mode Selection Gate and score each criterion before suggesting a mode.

## When to Run

After brainstorm output is complete (or after Fast Lane assessment for Fast Lane tasks).
This gate is MANDATORY. It cannot be skipped except when Fast Lane marks brainstorm as skipped —
in that case, run this gate with the Fast Lane assessment as input.

## Scoring Criteria

Score each criterion from the brainstorm output. Add up Mode B signals.

### Criterion 1: Domain Count
- 1 domain (e.g., only .NET, or only React) → Mode A signal
- 2+ domains (e.g., .NET + React, React Native + IoT) → Mode B signal

### Criterion 2: Risk Level
- Low risk: isolated change, no production data, easy rollback → Mode A signal
- Medium risk: affects shared services or APIs → neutral
- High risk: affects auth, data migration, production infra, security → Mode B signal

### Criterion 3: QA / DevOps Gate Needed
- No formal QA gate needed, reviewer sign-off is enough → Mode A signal
- Formal QA review with severity classification required → Mode B signal
- CI/CD pipeline design or deployment strategy needed → Mode B signal

### Criterion 4: Cross-Team Coordination
- Single owner, no handoff needed → Mode A signal
- Multiple roles needed (architect + implementer + QA + DevOps) → Mode B signal

### Criterion 5: Output Format Required
- Informal, conversational outputs sufficient → Mode A signal
- Machine-readable JSON contracts required (for CI, automation, audit) → Mode B signal

## Threshold Rules

- 0–1 Mode B signals → suggest Mode A
- 2 Mode B signals → suggest Mode A with note that Mode B is viable
- 3+ Mode B signals → suggest Mode B

## Hard Exclusions (Always Mode B)

These cases require Mode B regardless of other scores:
- Task involves 3+ technical domains
- Task requires a formal release gate (CI/CD pipeline sign-off)
- Task has compliance, security audit, or data migration impact
- Task requires explicit architecture decision records (ADRs)
- Task involves multiple implementer agents working in parallel

## Mode Selection Gate Output Format

Present this to the user after scoring:

```
Mode Selection Evaluation
─────────────────────────
Domain count:             [1 / 2 / 3+]       → [Mode A / Mode B] signal
Risk level:               [low / medium / high] → [Mode A / neutral / Mode B] signal
QA/DevOps gate needed:    [yes / no]          → [Mode B / Mode A] signal
Cross-team coordination:  [yes / no]          → [Mode B / Mode A] signal
Output format required:   [formal / informal] → [Mode B / Mode A] signal

Mode B signals: [N] / 5
Suggested mode: [A / B]
Reason: [one sentence citing the dominant signals]

Do you approve Mode [A/B], or would you like to override?
```

## Override Rule

User decision always wins. Once the user approves or overrides:
- Their chosen mode is the source of truth for ALL downstream phases.
- Claude does not re-evaluate or adjust after user approval.
- Spec, Plan, Implementation, QA, and Finish all follow the user-approved mode.
```

- [ ] **Step 2: Verify file was created**

Read `docs/claude/mode-selection-criteria.md` and confirm all sections are present:
- Scoring Criteria (5 criteria)
- Threshold Rules
- Hard Exclusions
- Mode Selection Gate Output Format
- Override Rule

- [ ] **Step 3: Commit**

```bash
git add docs/claude/mode-selection-criteria.md
git commit -m "feat: add mode-selection-criteria.md with scoring matrix and gate format"
```

---

### Task 2: Update CLAUDE.md — Execution Rule

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Read CLAUDE.md to locate the execution rule block**

Find the section starting with:
```
Execution rule in this workspace:
- Select runtime mode first (`Mode A: Solo`, `Mode B: Team Spine`).
```

- [ ] **Step 2: Replace the execution rule block**

Replace the entire `Execution rule in this workspace:` list with:

```markdown
Execution rule in this workspace:
- Run Brainstorm phase first (`skills/brainstorming/SKILL.md`) for all non-Fast-Lane tasks.
- After brainstorm, run Mode Selection Gate: read `docs/claude/mode-selection-criteria.md`, score each criterion, present suggested mode with reason, and wait for user approval.
- User-approved mode is the source of truth for all downstream phases. Do not re-evaluate after approval.
- In Mode A: run pure Superpowers workflow — solo spec, solo plan, solo implementation.
- In Mode B: Spec and Plan are written by team agents (phase-discovery-lead, phase-architecture-lead, phase-implementation-lead) using brainstorm output as input artifact. Orchestrator reviews internally, then user approves both spec and plan before implementation begins.
- In both modes, every implementation task must follow mandatory stack/domain skills from `docs/claude/stack-skill-rule-map.md` (example: C# tasks must follow `skills/implementer-dotnet-csharp/SKILL.md`).
- For hotfix/small tasks: run Fast Lane analysis (`fast-lane-assessment-v1`). If eligible, skip brainstorm but still run Mode Selection Gate with Fast Lane output as input. Minimum verification/testing gates are still required.
```

- [ ] **Step 3: Verify the edit**

Read the updated block and confirm:
- "Select runtime mode first" is gone
- "Run Brainstorm phase first" is the new first rule
- Mode Selection Gate rule is present
- User-approved mode rule is present
- Mode B team agents for spec/plan is mentioned
- Fast Lane rule still present

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "feat: update execution rule to brainstorm-first with mode selection gate"
```

---

### Task 3: Update docs/claude/current-process-workflow.md

**Files:**
- Modify: `docs/claude/current-process-workflow.md`

- [ ] **Step 1: Replace the Mode Entry section**

Find:
```markdown
## Mode Entry

Before execution starts, select one runtime mode:
- `Mode A: Superpowers Solo Mode` (no team-agent dependency)
- `Mode B: AI Team Spine Mode` (dispatcher + specialist/implementer agents)

Reference: `docs/claude/runtime-modes.md`
```

Replace with:
```markdown
## Workflow Entry

Before execution starts:

1. Run Fast Lane eligibility check (see Fast Lane Entry below).
2. If not Fast Lane: run Brainstorm phase — `skills/brainstorming/SKILL.md`.
3. Run Mode Selection Gate — read `docs/claude/mode-selection-criteria.md`, score complexity, present suggested mode, wait for user approval.
4. Execute the user-approved mode path below.

Reference: `docs/claude/runtime-modes.md`, `docs/claude/mode-selection-criteria.md`
```

- [ ] **Step 2: Replace the Fast Lane Entry section**

Find:
```markdown
## Fast Lane Entry (Hotfix/Small Task)

Before running the full phase flow, run Fast Lane analysis.
If the task is eligible, recommend skipping:
- Brainstorm phase
- Keep minimum verification/testing gate

Use template:
- `fast-lane-assessment-v1`
```

Replace with:
```markdown
## Fast Lane Entry (Hotfix/Small Task)

Run Fast Lane analysis before the full phase flow.
If eligible:
- Skip brainstorm phase
- Still run Mode Selection Gate (abbreviated, using Fast Lane output as input)
- Keep minimum verification/testing gate

Use template:
- `fast-lane-assessment-v1`
```

- [ ] **Step 3: Replace Mode A Workflow steps 1 and 2**

Find:
```markdown
## Mode A Workflow (Superpowers Solo Mode)

1. Session bootstrap phase - using `skills/using-superpowers/SKILL.md` - output:
- Skill-first execution is active
- Relevant skills are selected before implementation

2. Brainstorm phase - using `skills/brainstorming/SKILL.md` - output:
- Problem statement, scope, constraints, success criteria
- 2-3 approaches with trade-offs
- Approved design direction
- Skip when Fast Lane is approved
```

Replace with:
```markdown
## Mode A Workflow (Superpowers Solo Mode)

1. Session bootstrap phase - using `skills/using-superpowers/SKILL.md` - output:
- Skill-first execution is active
- Relevant skills are selected before implementation

2. Brainstorm phase - using `skills/brainstorming/SKILL.md` - output:
- Problem statement, scope, constraints, success criteria
- 2-3 approaches with trade-offs
- Approved design direction
- Skip when Fast Lane is approved

2b. Mode Selection Gate - using `docs/claude/mode-selection-criteria.md` - output:
- Scored complexity evaluation (5 criteria)
- Suggested mode with reason
- User-approved mode (Mode A confirmed)
```

- [ ] **Step 4: Replace Mode B Workflow steps 1 through 5**

Find:
```markdown
## Mode B Workflow (AI Team Spine Mode)

1. Session bootstrap phase - using `skills/using-superpowers/SKILL.md` - output:
- Skill-first behavior is active
- Combined workflow is ready

2. Brainstorm and design phase - using `skills/brainstorming/SKILL.md` - output:
- Approved design direction
- Written and approved spec document
- Skip when Fast Lane is approved

3. Planning phase - using `skills/writing-plans/SKILL.md` - output:
- Approved implementation plan with small tasks

4. Orchestration intake phase - using `agents/team-orchestrator.md` - output:
- Shared objective, ownership, risks, checkpoints
- `orchestrator-status-v1`

5. Dispatch phase - using `agents/master-dispatcher.md` and `docs/claude/master-dispatcher-prompt.md` - output:
- `task_type`, selected owner agent, phase, done criteria
- Output template assignment
- Dispatcher JSON contract
- Fast Lane recommendation via `fast-lane-assessment-v1`
```

Replace with:
```markdown
## Mode B Workflow (AI Team Spine Mode)

1. Session bootstrap phase - using `skills/using-superpowers/SKILL.md` - output:
- Skill-first behavior is active
- Combined workflow is ready

2. Brainstorm phase - using `skills/brainstorming/SKILL.md` - output:
- Problem statement, scope, constraints, success criteria
- 2-3 approaches with trade-offs
- Approved design direction (this becomes the input artifact for spec/plan)
- Skip when Fast Lane is approved

2b. Mode Selection Gate - using `docs/claude/mode-selection-criteria.md` - output:
- Scored complexity evaluation (5 criteria)
- Suggested mode with reason
- User-approved mode (Mode B confirmed)

3. Orchestration intake phase - using `agents/team-orchestrator.md` - output:
- Shared objective, ownership, risks, checkpoints
- Input: brainstorm output as context artifact
- `orchestrator-status-v1`

4. Dispatch phase - using `agents/master-dispatcher.md` and `docs/claude/master-dispatcher-prompt.md` - output:
- Routes spec task → phase-discovery-lead + phase-architecture-lead
- Routes plan task → phase-implementation-lead
- Dispatcher JSON contract

5. Discovery phase - using `agents/phase-discovery-lead.md` - output:
- Input: brainstorm output (no re-brainstorming)
- Formalizes requirements spec from brainstorm output
- `phase-lead-report-v1`

6. Architecture phase - using `agents/phase-architecture-lead.md` - output:
- Input: brainstorm output + discovery report
- Formalizes technical spec (contracts, interfaces, trade-offs)
- `phase-lead-report-v1` or `specialist-report-v1`

7. Spec review phase - output:
- Orchestrator internal review gate
- Spec written to `docs/specs/YYYY-MM-DD-design.md`
- User approves or requests changes

8. Planning phase - using `agents/phase-implementation-lead.md` - output:
- Input: user-approved spec
- Ordered, testable implementation plan
- `phase-lead-report-v1`

9. Plan review phase - output:
- Orchestrator internal review gate
- User approves or requests changes
```

- [ ] **Step 5: Renumber the remaining Mode B steps**

After inserting new steps 5-9, renumber the former steps 6-12 to become 10-16. Verify numbering is sequential.

- [ ] **Step 6: Verify the full file reads correctly**

Read `docs/claude/current-process-workflow.md` and confirm:
- Mode Entry section replaced with Workflow Entry
- Mode Selection Gate step (2b) exists in both Mode A and Mode B
- Mode B steps 5-9 cover spec/plan via team agents
- Brainstorm output as input artifact is mentioned
- All remaining steps are correctly renumbered

- [ ] **Step 7: Commit**

```bash
git add docs/claude/current-process-workflow.md
git commit -m "feat: update current-process-workflow with brainstorm-first and mode selection gate"
```

---

### Task 4: Update docs/claude/runtime-modes.md

**Files:**
- Modify: `docs/claude/runtime-modes.md`

- [ ] **Step 1: Replace the Mode Selection Rules section**

Find:
```markdown
## Mode Selection Rules

Choose Mode A when:
- Task scope is small to medium
- One technical domain is dominant
- You want minimal orchestration overhead

Choose Mode B when:
- Work spans multiple domains or stacks
- You need explicit QA and DevOps gating
- You want machine-readable, role-based reporting

## Default Recommendation

- Start with Mode A for straightforward tasks.
- Switch to Mode B when complexity, risk, or cross-team coordination increases.
```

Replace with:
```markdown
## Mode Selection Rules

Mode selection is determined by the Mode Selection Gate after brainstorming.

Reference: `docs/claude/mode-selection-criteria.md`

The criteria file contains:
- 5 scored criteria (domain count, risk, QA/DevOps need, cross-team coordination, output format)
- Threshold rules for mode suggestion
- Hard exclusions that always require Mode B
- Gate output format to present to user
- Override rule: user decision always wins

## Default Recommendation

Run the Mode Selection Gate after every brainstorm session. Do not select a mode manually without running the gate first.
```

- [ ] **Step 2: Verify**

Read the updated section and confirm the old inline rules are removed and the pointer to `mode-selection-criteria.md` is present.

- [ ] **Step 3: Commit**

```bash
git add docs/claude/runtime-modes.md
git commit -m "feat: remove inline mode selection rules, point to mode-selection-criteria.md"
```

---

### Task 5: Update docs/claude/master-dispatcher-prompt.md

**Files:**
- Modify: `docs/claude/master-dispatcher-prompt.md`

- [ ] **Step 1: Update the Input You Receive section**

Find:
```markdown
## Input You Receive
- User objective
- Constraints (time, quality, stack, compliance)
- Current phase (if known)
- Existing artifacts (spec, plan, code, tests, pipeline)
```

Replace with:
```markdown
## Input You Receive
- User objective
- Constraints (time, quality, stack, compliance)
- Current phase (if known)
- Existing artifacts (spec, plan, code, tests, pipeline)
- Brainstorm output artifact (problem, scope, constraints, approved design direction) — present for all non-Fast-Lane tasks
```

- [ ] **Step 2: Add spec/plan routing entries to the Routing Table**

Find the Routing Table section and add two entries after `implementation-planning`:

```markdown
- `spec-discovery` -> `phase-discovery-lead` (input: brainstorm output)
- `spec-architecture` -> `phase-architecture-lead` (input: brainstorm output + discovery report)
- `plan-writing` -> `phase-implementation-lead` (input: approved spec)
```

- [ ] **Step 3: Update the Dispatch Algorithm step 1**

Find:
```markdown
1. Normalize request into one dominant `task_type` and optional secondary task types.
```

Replace with:
```markdown
1. Normalize request into one dominant `task_type` and optional secondary task types.
   - If brainstorm output is present and task_type is spec or plan: use brainstorm output as primary input artifact. Do not brainstorm again.
```

- [ ] **Step 4: Verify**

Read and confirm:
- Brainstorm output listed in Input section
- spec-discovery, spec-architecture, plan-writing in routing table
- No re-brainstorm rule added to algorithm

- [ ] **Step 5: Commit**

```bash
git add docs/claude/master-dispatcher-prompt.md
git commit -m "feat: add brainstorm output input and spec/plan routing to master-dispatcher-prompt"
```

---

### Task 6: Update docs/claude/full-ai-team-setup.md

**Files:**
- Modify: `docs/claude/full-ai-team-setup.md`

- [ ] **Step 1: Update Section 7 (Suggested Execution Pattern)**

Find:
```markdown
## 7) Suggested Execution Pattern in Claude Code
1. Team Orchestrator receives request and dispatches phase lead
2. Phase lead creates scoped tasks and dispatches specialist/implementer agents
3. QA gate runs before phase completion
4. DevOps gate runs before release decisions
5. Orchestrator summarizes status, blockers, and next actions
```

Replace with:
```markdown
## 7) Suggested Execution Pattern in Claude Code
1. Brainstorm phase runs first — output: problem, scope, constraints, approved design direction
2. Mode Selection Gate runs — read `docs/claude/mode-selection-criteria.md`, score, suggest mode, user approves
3. (Mode B) Team Orchestrator receives brainstorm output as context and dispatches spec/plan leads
4. (Mode B) Discovery Lead + Architecture Lead formalize spec from brainstorm output (no re-brainstorm)
5. (Mode B) Orchestrator reviews spec internally, then user approves spec
6. (Mode B) Implementation Lead writes plan from approved spec
7. (Mode B) Orchestrator reviews plan internally, then user approves plan
8. Phase lead dispatches stack implementer agents for implementation
9. QA gate runs before phase completion
10. DevOps gate runs before release decisions
11. Orchestrator summarizes status, blockers, and next actions
```

- [ ] **Step 2: Update Section 8 to reference mode-selection-criteria.md**

Find:
```markdown
- Runtime modes: `docs/claude/runtime-modes.md`
```

Add after it:
```markdown
- Mode selection criteria: `docs/claude/mode-selection-criteria.md`
```

- [ ] **Step 3: Verify**

Read and confirm Section 7 shows brainstorm-first pattern and Section 8 references criteria file.

- [ ] **Step 4: Commit**

```bash
git add docs/claude/full-ai-team-setup.md
git commit -m "feat: update full-ai-team-setup execution pattern to brainstorm-first"
```

---

### Task 7: Update skills/brainstorming/SKILL.md

**Files:**
- Modify: `skills/brainstorming/SKILL.md`

- [ ] **Step 1: Add Mode Selection Gate to the Checklist**

Find the Checklist section:
```markdown
## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — check files, docs, recent commits
2. **Offer visual companion** (if topic will involve visual questions) ...
3. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
4. **Propose 2-3 approaches** — with trade-offs and your recommendation
5. **Present design** — in sections scaled to their complexity, get user approval after each section
6. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
8. **User reviews written spec** — ask user to review the spec file before proceeding
9. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

Replace with:
```markdown
## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — check files, docs, recent commits
2. **Offer visual companion** (if topic will involve visual questions) ...
3. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
4. **Propose 2-3 approaches** — with trade-offs and your recommendation
5. **Present design** — in sections scaled to their complexity, get user approval after each section
6. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
8. **User reviews written spec** — ask user to review the spec file before proceeding
9. **Mode Selection Gate** — read `docs/claude/mode-selection-criteria.md`, score complexity, present suggested mode with reason, wait for user approval (see Mode Selection Gate section below)
10. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

- [ ] **Step 2: Add Mode Selection Gate section to the skill**

Append the following section before the Visual Companion section:

```markdown
## Mode Selection Gate

This gate runs at the end of every brainstorm session (step 9 of the checklist), before invoking writing-plans.

**Steps:**
1. Read `docs/claude/mode-selection-criteria.md`
2. Score each of the 5 criteria against the brainstorm output
3. Count Mode B signals and determine suggestion per threshold rules
4. Check hard exclusions — if any apply, suggest Mode B regardless of score
5. Present the gate output in the format defined in `mode-selection-criteria.md`
6. Wait for user to approve or override the suggested mode

**Rule:** Do not proceed to writing-plans until user has approved a mode.

**User authority:** User-approved mode is the source of truth for all downstream phases. If user overrides from Mode B to Mode A, all subsequent phases (spec, plan, implementation, QA, finish) run in Mode A — no exceptions.

**Fast Lane exception:** If this session started via Fast Lane (brainstorm was skipped), run the gate using the Fast Lane output (`fast-lane-assessment-v1`) as input instead of brainstorm output.
```

- [ ] **Step 3: Verify**

Read the updated skill and confirm:
- Checklist item 9 is Mode Selection Gate
- Former item 9 (Transition) is now item 10
- Mode Selection Gate section exists with steps and rules

- [ ] **Step 4: Commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: add Mode Selection Gate to brainstorming skill checklist"
```

---

### Task 8: Update skills/writing-plans/SKILL.md

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

- [ ] **Step 1: Add Mode B path note to the Overview section**

Find:
```markdown
**Context:** This should be run in a dedicated worktree (created by brainstorming skill).
```

Add after it:
```markdown
**Mode B path:** When invoked after a Mode B session, the implementation plan input is the user-approved spec written by team agents (phase-discovery-lead + phase-architecture-lead). The plan is written by phase-implementation-lead. This skill guides the plan structure regardless of who writes it.
```

- [ ] **Step 2: Verify**

Read and confirm Mode B path note is present in the Context section.

- [ ] **Step 3: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat: add Mode B path note to writing-plans skill"
```

---

### Task 9: Update agents/team-orchestrator.md

**Files:**
- Modify: `agents/team-orchestrator.md`

- [ ] **Step 1: Add brainstorm context and spec/plan gate to Responsibilities**

Find:
```markdown
Responsibilities:
1. Convert user goals into phase-based execution.
2. Dispatch the right specialist or implementer agent for each task.
3. Keep a single source of truth for scope, assumptions, and risks.
4. Enforce quality gates: QA review and CI/CD readiness before completion.
5. Return concise status: done, in-progress, blocked, next action.
6. Route work using `docs/claude/master-dispatcher-prompt.md`.
7. Require agent outputs to follow `docs/claude/agent-output-templates.md`.
```

Replace with:
```markdown
Responsibilities:
1. Receive brainstorm output as context artifact — do not re-brainstorm.
2. Convert approved design direction into phase-based execution.
3. Dispatch the right specialist or implementer agent for each task.
4. Own the spec/plan review gate in Mode B:
   a. Review spec produced by Discovery Lead + Architecture Lead (internal gate).
   b. Present spec to user and wait for approval before proceeding.
   c. Review implementation plan produced by Implementation Lead (internal gate).
   d. Present plan to user and wait for approval before implementation begins.
5. Keep a single source of truth for scope, assumptions, and risks.
6. Enforce quality gates: QA review and CI/CD readiness before completion.
7. Return concise status: done, in-progress, blocked, next action.
8. Route work using `docs/claude/master-dispatcher-prompt.md`.
9. Require agent outputs to follow `docs/claude/agent-output-templates.md`.
```

- [ ] **Step 2: Verify**

Read and confirm:
- Responsibility 1 is "Receive brainstorm output as context artifact"
- Responsibility 4 describes the spec/plan review gate with user approval steps

- [ ] **Step 3: Commit**

```bash
git add agents/team-orchestrator.md
git commit -m "feat: add brainstorm context and spec/plan review gate to team-orchestrator"
```

---

### Task 10: Update agents/master-dispatcher.md

**Files:**
- Modify: `agents/master-dispatcher.md`

- [ ] **Step 1: Add spec/plan routing to Core behavior**

Find:
```markdown
Core behavior:
- Follow `docs/claude/master-dispatcher-prompt.md` for routing logic.
- Select one primary agent owner for each task.
- Include secondary agents only when cross-phase collaboration is required.
- Require output template IDs from `docs/claude/agent-output-templates.md`.
```

Replace with:
```markdown
Core behavior:
- Follow `docs/claude/master-dispatcher-prompt.md` for routing logic.
- Select one primary agent owner for each task.
- Include secondary agents only when cross-phase collaboration is required.
- Require output template IDs from `docs/claude/agent-output-templates.md`.
- For spec tasks: route to `phase-discovery-lead` (requirements) and `phase-architecture-lead` (technical). Pass brainstorm output as input artifact. These agents formalize the spec — they do not brainstorm again.
- For plan tasks: route to `phase-implementation-lead`. Pass user-approved spec as input artifact.
```

- [ ] **Step 2: Verify**

Read and confirm spec/plan routing rules are present with brainstorm output as input.

- [ ] **Step 3: Commit**

```bash
git add agents/master-dispatcher.md
git commit -m "feat: add spec/plan routing rules to master-dispatcher"
```

---

### Task 11: Update agents/phase-discovery-lead.md

**Files:**
- Modify: `agents/phase-discovery-lead.md`

- [ ] **Step 1: Add input constraint and spec formalization task**

Find:
```markdown
You lead Phase 1 (Discovery and Scope).

Goals:
- Clarify business and technical requirements.
- Identify constraints (timeline, resources, compatibility, compliance).
- Define acceptance criteria and non-goals.
- Produce risk register with first mitigations.

Deliverables:
- Problem statement
- In-scope vs out-of-scope
- Acceptance criteria (testable)
- Risk list (likelihood x impact)
- Open questions that block architecture
```

Replace with:
```markdown
You lead Phase 1 (Discovery and Scope).

Input:
- Brainstorm output artifact (problem, scope, constraints, approved design direction).
- You MUST use this as your primary input. Do NOT brainstorm again from scratch.
- Your job is to formalize and structure the brainstorm output into a requirements spec.

Goals:
- Formalize business and technical requirements from brainstorm output.
- Identify and document constraints (timeline, resources, compatibility, compliance).
- Define testable acceptance criteria and explicit non-goals.
- Produce risk register with first mitigations.

Deliverables:
- Requirements spec written to `docs/specs/YYYY-MM-DD-design.md` (requirements section)
- Problem statement (from brainstorm output, formalized)
- In-scope vs out-of-scope
- Acceptance criteria (testable)
- Risk list (likelihood x impact)
- Open questions that block architecture
```

- [ ] **Step 2: Verify**

Read and confirm:
- Input section is present at the top
- "Do NOT brainstorm again from scratch" rule is present
- Deliverables include spec file path

- [ ] **Step 3: Commit**

```bash
git add agents/phase-discovery-lead.md
git commit -m "feat: add brainstorm input constraint and spec formalization to phase-discovery-lead"
```

---

### Task 12: Update agents/phase-architecture-lead.md

**Files:**
- Modify: `agents/phase-architecture-lead.md`

- [ ] **Step 1: Add input constraint and spec formalization task**

Find:
```markdown
You lead Phase 2 (Architecture and Design).

Goals:
- Define target architecture and module boundaries.
- Specify APIs/contracts, data flow, and state ownership.
- Evaluate trade-offs (performance, complexity, maintainability).
- Produce migration and rollout strategy.

Deliverables:
- Architecture summary
- Decision log (ADR-style)
- Interface contracts and failure modes
- Rollout and rollback approach
- Technical risks and mitigations
```

Replace with:
```markdown
You lead Phase 2 (Architecture and Design).

Input:
- Brainstorm output artifact (problem, scope, constraints, approved design direction).
- Discovery report from phase-discovery-lead (`phase-lead-report-v1`).
- You MUST use these as your primary input. Do NOT brainstorm again from scratch.
- Your job is to formalize the technical design from the approved direction into a technical spec.

Goals:
- Define target architecture and module boundaries from brainstorm output.
- Specify APIs/contracts, data flow, and state ownership.
- Evaluate trade-offs (performance, complexity, maintainability).
- Produce migration and rollout strategy.

Deliverables:
- Technical spec written to `docs/specs/YYYY-MM-DD-design.md` (architecture section)
- Architecture summary
- Decision log (ADR-style)
- Interface contracts and failure modes
- Rollout and rollback approach
- Technical risks and mitigations
```

- [ ] **Step 2: Verify**

Read and confirm Input section is present, no re-brainstorm rule exists, spec file path in deliverables.

- [ ] **Step 3: Commit**

```bash
git add agents/phase-architecture-lead.md
git commit -m "feat: add brainstorm input constraint and technical spec ownership to phase-architecture-lead"
```

---

### Task 13: Update agents/phase-implementation-lead.md

**Files:**
- Modify: `agents/phase-implementation-lead.md`

- [ ] **Step 1: Add input constraint and plan writing responsibility**

Find:
```markdown
You lead Phase 3 (Implementation).

Goals:
- Break design into small tasks with clear ownership.
- Dispatch stack implementers by language/framework fit.
- Require tests and verification for each increment.
- Keep change log and integration notes.

Deliverables:
- Task board with ownership
- Incremental code changes
- Test evidence per task
- Integration status and blockers
```

Replace with:
```markdown
You lead Phase 3 (Implementation).

Input:
- User-approved spec from Phase 1 (requirements) and Phase 2 (architecture).
- Your first responsibility is to write the implementation plan from this approved spec before dispatching implementers.

Goals:
- Write an ordered, testable implementation plan from the approved spec.
- Break design into small tasks with clear ownership.
- Dispatch stack implementers by language/framework fit.
- Require tests and verification for each increment.
- Keep change log and integration notes.

Plan writing:
- Follow `skills/writing-plans/SKILL.md` structure.
- Submit plan to team-orchestrator for internal review.
- Wait for user approval before dispatching any implementer.

Deliverables:
- Implementation plan (submitted to orchestrator for review, then user-approved)
- Task board with ownership
- Incremental code changes
- Test evidence per task
- Integration status and blockers
```

- [ ] **Step 2: Verify**

Read and confirm:
- Input section is present
- Plan writing responsibility is explicit
- "Wait for user approval before dispatching" is present

- [ ] **Step 3: Commit**

```bash
git add agents/phase-implementation-lead.md
git commit -m "feat: add spec input and plan writing responsibility to phase-implementation-lead"
```

---

## Final Verification

- [ ] **Read docs/claude/mode-selection-criteria.md** — confirm all 5 criteria, threshold rules, hard exclusions, gate format, override rule
- [ ] **Read CLAUDE.md execution rule** — confirm brainstorm-first, mode gate, user authority, Mode B team agents for spec/plan
- [ ] **Read docs/claude/current-process-workflow.md** — confirm workflow entry, step 2b in both modes, Mode B steps 5-9
- [ ] **Check git log** — confirm 13 commits, one per task

```bash
git log --oneline -15
```

- [ ] **Final commit if any loose files remain**

```bash
git status
```
