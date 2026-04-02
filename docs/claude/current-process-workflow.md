# Current Process Workflow (Superpowers + Personalized AI Team Plugin)

This file records the complete runtime workflow for the personalized plugin.

## Mode Entry

Before execution starts, select one runtime mode:
- `Mode A: Superpowers Solo Mode` (no team-agent dependency)
- `Mode B: AI Team Spine Mode` (dispatcher + specialist/implementer agents)

Reference: `docs/claude/runtime-modes.md`

## Fast Lane Entry (Hotfix/Small Task)

Before running the full phase flow, run Fast Lane analysis.
If the task is eligible, recommend skipping:
- Brainstorm phase
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

6. Discovery/architecture alignment phase - using `agents/phase-discovery-lead.md` and `agents/phase-architecture-lead.md` (plus specialists as needed) - output:
- Requirement and acceptance alignment
- Architecture/contracts/trade-offs alignment
- `phase-lead-report-v1` or `specialist-report-v1`

7. Implementation control phase - using `agents/phase-implementation-lead.md` - output:
- Task ownership and implementation sequencing
- Integration status and blockers
- `phase-lead-report-v1`

8. Stack/domain implementation phase - using dispatcher-selected implementer agent:
- `agents/implementer-react-native-typescript.md`
- `agents/implementer-dotnet-csharp.md`
- `agents/implementer-angular-typescript.md`
- `agents/implementer-react-typescript.md`
- `agents/implementer-iot-edge.md`
- output:
- Changed files, tests, quality checks, risks
- `implementer-delivery-v1`
- Mandatory stack skill source: `docs/claude/stack-skill-rule-map.md`

9. QA gate phase - using `agents/phase-qa-gate.md`, `agents/qa-code-reviewer.md`, and `skills/requesting-code-review/SKILL.md` - output:
- Findings by severity
- Coverage/regression assessment
- Release recommendation
- `qa-review-v1`

10. Release and operations phase - using `agents/phase-release-devops-lead.md`, `agents/devops-cicd-assistant.md`, and `skills/finishing-a-development-branch/SKILL.md` - output:
- CI/CD gates, deployment strategy, rollback, observability
- Release readiness decision
- `devops-release-v1` plus phase summary

11. Final handoff phase - using `agents/team-orchestrator.md` and `docs/claude/agent-output-templates.md` - output:
- Final status, blockers, residual risks, next actions
- `orchestrator-status-v1`

12. Escalation phase - using `docs/claude/master-dispatcher-prompt.md` - output:
- 2-3 option escalation when constraints conflict with scope, architecture, security/reliability, or timeline

---

## Output Standard

- Mode A: follow outputs required by each Superpowers skill.
- Mode B: every agent output must include a valid `template_id` from `docs/claude/agent-output-templates.md`.
- Both modes: each implementation task must follow the stack/domain skill from `docs/claude/stack-skill-rule-map.md`.
- Fast Lane: use `fast-lane-assessment-v1` and recommend skipping brainstorm only when eligibility is true.
