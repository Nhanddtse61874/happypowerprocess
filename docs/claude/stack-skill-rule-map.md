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

## How to Apply in Mode A (Solo)

Mode A does not require spawning team agents.
However, the active coding agent must still load and obey the relevant stack skill before implementation.

Example:
- If task domain is C# backend, mandatory skill is `skills/implementer-dotnet-csharp/SKILL.md`.

## How to Apply in Mode B (Team Spine)

Dispatcher selects the implementer agent by `task_type` and stack.
Selected implementer agent is responsible for execution ownership.
The matching stack skill remains mandatory for execution and review.
