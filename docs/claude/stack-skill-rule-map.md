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
   - If found and `skill_path` begins with `.claude/...` → load the project snapshot (overrides plugin default).
   - If found and `source == "plugin"` and the path lives in the plugin install dir → load the plugin default listed above.
   - If not found → halt and instruct the user to run `/add-tech-stack <stack>` (T2 trigger).

Project snapshots always win over plugin defaults when both exist. The 5 plugin defaults remain available as fallbacks until the user customizes them.

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
