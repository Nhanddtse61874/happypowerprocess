---
description: "Bootstrap project state files, config, CLAUDE.md, permissions, and gitignore for happypowerprocess workflow"
---

Invoke the **happypowerprocess:init-project** skill to bootstrap this project.

The skill handles:
- State files (`.planning/PROJECT.md`, `.planning/REQUIREMENTS.md`, `.planning/ROADMAP.md`, `.planning/STATE.md`)
- Workflow config (`.planning/config.json`)
- Project-level `CLAUDE.md` (from plugin template)
- Permission allowlist (`.claude/settings.json` + `.claude/settings.local.json`)
- `.gitignore` entries for workflow artifacts
- Optional git init + initial commit prompts

Asks 2 minimal prompts: project name + primary stack. Auto-detects brownfield via Glob markers (`*.csproj`, `package.json`, `Cargo.toml`, etc.).

**Idempotency**: If project already initialized (STATE.md exists), shows 4-choice menu (abort / update config / re-init missing / force re-init with backup).

**`--force` flag**: If the user message contains `--force` (case-insensitive), skip the menu and jump directly to force re-init with 1 confirmation.

Pass the user's full invocation message to the skill so it can detect flags and modifiers.
