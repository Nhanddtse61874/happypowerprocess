# Stack Skill and Rule Map

This map defines mandatory stack-specific guidance for every implementation task.

## Enforcement Rule

For every implementation task, the agent must do both:
1. Follow Superpowers execution skills
2. Follow the stack/domain skill listed below

This rule applies in both runtime modes:
- Mode A (Solo)
- Mode B (AI Team Spine)

## Stack to Mandatory Skill Mapping

- .NET / C# backend:
  - `skills/implementer-dotnet-csharp/SKILL.md`
- React Native / TypeScript:
  - `skills/implementer-react-native-typescript/SKILL.md`
- Angular / TypeScript:
  - `skills/implementer-angular-typescript/SKILL.md`
- React / TypeScript:
  - `skills/implementer-react-typescript/SKILL.md`
- IoT edge (MQTT/BLE/device integration):
  - `skills/implementer-iot-edge/SKILL.md`

## Runtime Lookup Order (v5.5.0+)

When the agent needs the skill or agent for a stack:

1. Read `.claude/stack-skills/registry.json` (project-scoped registry, source of truth).
2. Find the entry where `name == <stack>`:
   - If found: load skill from `entry.skill_path` and agent from `entry.agent_path`. The registry path field is authoritative.
     - For `source == "plugin"` entries, paths point to plugin install dir (`skills/implementer-<canonical>/SKILL.md`).
     - For `source == "customized"` or `source == "project"` entries, paths point to `.claude/...` snapshot files in the project. The registry path field is rewritten by `/add-tech-stack` whenever source flips, so this branch always reflects the current customization state.
   - If not found → halt and instruct the user to run `/add-tech-stack <stack>` (T2 trigger).

Project snapshots always win over plugin defaults: when `/add-tech-stack` customizes a stack, it (a) writes the snapshot files, (b) flips `source` to `customized`, AND (c) rewrites `skill_path` + `agent_path` to point to the snapshots. The 5 plugin defaults remain available unchanged at the plugin install location and serve as fallback if a user manually deletes their snapshot directory and resets `source` back to `plugin`.

## How to Apply in Mode A (Solo)

Mode A does not require spawning team agents.
However, the active coding agent must still load and obey the relevant stack skill before implementation.

Example:
- If task domain is C# backend, mandatory skill is `skills/implementer-dotnet-csharp/SKILL.md`.

## How to Apply in Mode B (Team Spine)

Dispatcher selects the implementer agent by `task_type` and stack.
Selected implementer agent is responsible for execution ownership.
The matching stack skill remains mandatory for execution and review.

## Unknown Stack Behavior (T2)

If the agent receives a task that mentions a stack not present in `.claude/stack-skills/registry.json`, the agent MUST halt and present these two options:

```
⚠ Stack '<name>' not registered in this project's registry.json.
I cannot proceed without a stack skill.
💡 Suggestions:
  [1] Run /add-tech-stack <name>  (interactive)
  [2] Cancel task
```

The agent does **not** auto-create the stack. The user must explicitly invoke `/add-tech-stack` (E3 conservative — see spec §7.5).

Detection heuristic: scan the task text for known framework keywords (FastAPI, Fiber, Axum, Spring, etc.) and for file globs from `detection_markers` of registered stacks. False positives prefer asking over over-triggering.
