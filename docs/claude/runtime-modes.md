# Runtime Modes (Personalized Plugin)

This plugin supports two runtime modes.

## Mode A: Superpowers Solo Mode (No Team Agents Required)

Use this mode when you want a lighter workflow without agent-team orchestration.

Execution backbone:
- `skills/using-superpowers/SKILL.md`
- `skills/brainstorming/SKILL.md`
- `skills/writing-plans/SKILL.md`
- `skills/subagent-driven-development/SKILL.md` or `skills/executing-plans/SKILL.md`
- `skills/test-driven-development/SKILL.md`
- `skills/requesting-code-review/SKILL.md`
- `skills/finishing-a-development-branch/SKILL.md`

Characteristics:
- No dependency on `agents/*.md` team roles
- Faster setup for small/medium tasks
- Strong process discipline from Superpowers skills
- Mandatory stack/domain rule adherence (see `docs/claude/stack-skill-rule-map.md`)

## Mode B: AI Team Spine Mode (Superpowers + Agent Team)

Use this mode when you want role-based execution and explicit ownership across phases.

Execution backbone:
- All Superpowers execution skills from Mode A
- Team orchestration layer:
  - `agents/team-orchestrator.md`
  - `agents/master-dispatcher.md`
  - `docs/claude/master-dispatcher-prompt.md`
  - `docs/claude/agent-output-templates.md`
- Specialist and implementer agents selected by dispatcher

Characteristics:
- Clear role ownership and phase gates
- Structured outputs with template IDs
- Better control for larger and cross-domain work

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

## Output Contract

- Mode A: Superpowers workflow outputs as defined by each skill.
- Mode B: Must use `template_id` contracts in `docs/claude/agent-output-templates.md`.

## Mandatory Stack Rule Enforcement (Both Modes)

For every implementation task, stack/domain skills are mandatory even in Mode A.

Example:
- C# backend task must follow `skills/implementer-dotnet-csharp/SKILL.md`.

Reference:
- `docs/claude/stack-skill-rule-map.md`

## Fast Lane (Hotfix/Small Task) - Both Modes

Both modes support a Fast Lane recommendation path for hotfix and small tasks.

If Fast Lane analysis marks the task as eligible, the system should recommend:
- Skip brainstorm phase
- Keep minimum verification/testing gate

Conditions:
- The required change is explicit and low-risk
- No architecture/API change
- No security-sensitive or migration risk

Fast Lane output template:
- `fast-lane-assessment-v1` from `docs/claude/agent-output-templates.md`
