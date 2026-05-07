---
description: "Customize an existing plugin stack skill for current project, or add a new stack not shipped with plugin"
---

Invoke the **happypowerprocess:add-tech-stack** skill to manage stack skills for this project.

The skill handles two scenarios:
- **Scenario A — existing stack** (registered in `.claude/stack-skills/registry.json`): walks each section with `[keep / override / append / skip]`. User picks file scope: SKILL only / AGENT only / both.
- **Scenario B — new stack** (not registered): forces both SKILL.md + AGENT.md. Walks template sections with `[input / suggest / skip]`. Best-practice MINIMUM filled on explicit skip. Variable section loop for stack-specific patterns.

**Architecture detection**: opt-in heuristic scan of project structure. Triggered in (a) `/init-project` brownfield T1 sub-flow and (b) the `suggest` action of `/add-tech-stack` Scenario B `## Architecture` section walk. Scenario A actions (`keep / override / append / skip`) do not trigger detection. E3 conservative — every step requires explicit user consent. Web search always opt-in via prompt; no `--use-web` flag.

**Snapshot model**: writes to `.claude/stack-skills/<stack>/SKILL.md` and `.claude/agents/implementer-<stack>.md`. Project snapshot wins over plugin default at runtime.

**Idempotency**: If stack already customized in project, shows menu `[continue overwrite / merge / cancel]`. `merge` re-routes to `/sync-stack-skill <name>`.

**`--force` flag**: If the user message contains `--force` (case-insensitive), skip the file-choice prompt in Scenario A → force walks both SKILL+AGENT with single backup confirmation.

Pass the user's full invocation message to the skill so it can detect arguments (stack name) and flags (`--force`).
