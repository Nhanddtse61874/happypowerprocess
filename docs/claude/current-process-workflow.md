# Current Process Workflow (Superpowers + Personalized AI Team Plugin)

This file records the complete runtime workflow for the personalized plugin.

## Workflow Entry

Before execution starts:

1. Run Fast Lane eligibility check (see Fast Lane Entry below).
2. If not Fast Lane: run Brainstorm phase — `skills/brainstorming/SKILL.md`.
3. Run Mode Selection Gate — read `docs/claude/mode-selection-criteria.md`, score complexity, present suggested mode, wait for user approval.
4. Execute the user-approved mode path below.

Reference: `docs/claude/runtime-modes.md`, `docs/claude/mode-selection-criteria.md`

## Fast Lane Entry (Hotfix/Small Task)

Run Fast Lane analysis before the full phase flow.
If eligible:
- Skip brainstorm phase
- Still run Mode Selection Gate (abbreviated, using Fast Lane output as input)
- Keep minimum verification/testing gate

Use template:
- `fast-lane-assessment-v1`

---

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

3. Design documentation phase - using `skills/brainstorming/SKILL.md` - output:
- Spec file in `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
- Self-reviewed and user-approved spec

4. Planning phase - using `skills/writing-plans/SKILL.md` - output:
- Ordered, testable implementation plan
- Verification steps per task

5. Implementation phase - using `skills/subagent-driven-development/SKILL.md` or `skills/executing-plans/SKILL.md` - output:
- Task-by-task implementation
- Review loops and progress tracking
- Mandatory stack skill is applied per task (see `docs/claude/stack-skill-rule-map.md`)

6. Test discipline phase - using `skills/test-driven-development/SKILL.md` - output:
- RED-GREEN-REFACTOR evidence
- Passing tests for implemented behavior
- Minimum verification/testing gate is still required in Fast Lane

7. QA phase - using `skills/requesting-code-review/SKILL.md` - output:
- Severity-based findings
- Fix list and residual risk notes

8. Finish phase - using `skills/finishing-a-development-branch/SKILL.md` - output:
- Final verification
- Branch completion decision (merge/PR/keep/discard)

---

## Mode B Workflow (AI Team Spine Mode)

1. Session bootstrap phase - using `skills/using-superpowers/SKILL.md` - output:
- Skill-first behavior is active
- Combined workflow is ready

2. Brainstorm phase - using `skills/brainstorming/SKILL.md` - output:
- Problem statement, scope, constraints, success criteria
- 2-3 approaches with trade-offs
- Approved design direction (becomes input artifact for spec/plan)
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

10. Stack/domain implementation phase - using dispatcher-selected implementer agent:
- `agents/implementer-react-native-typescript.md`
- `agents/implementer-dotnet-csharp.md`
- `agents/implementer-angular-typescript.md`
- `agents/implementer-react-typescript.md`
- `agents/implementer-iot-edge.md`
- output:
- Changed files, tests, quality checks, risks
- `implementer-delivery-v1`
- Mandatory stack skill source: `docs/claude/stack-skill-rule-map.md`

11. QA gate phase - using `agents/phase-qa-gate.md`, `agents/qa-code-reviewer.md`, and `skills/requesting-code-review/SKILL.md` - output:
- Findings by severity
- Coverage/regression assessment
- Release recommendation
- `qa-review-v1`

12. Release and operations phase - using `agents/phase-release-devops-lead.md`, `agents/devops-cicd-assistant.md`, and `skills/finishing-a-development-branch/SKILL.md` - output:
- CI/CD gates, deployment strategy, rollback, observability
- Release readiness decision
- `devops-release-v1` plus phase summary

13. Final handoff phase - using `agents/team-orchestrator.md` and `docs/claude/agent-output-templates.md` - output:
- Final status, blockers, residual risks, next actions
- `orchestrator-status-v1`

14. Escalation phase - using `docs/claude/master-dispatcher-prompt.md` - output:
- 2-3 option escalation when constraints conflict with scope, architecture, security/reliability, or timeline

---

## Output Standard

- Mode A: follow outputs required by each Superpowers skill.
- Mode B: every agent output must include a valid `template_id` from `docs/claude/agent-output-templates.md`.
- Both modes: each implementation task must follow the stack/domain skill from `docs/claude/stack-skill-rule-map.md`.
- Fast Lane: use `fast-lane-assessment-v1` and recommend skipping brainstorm only when eligibility is true.
