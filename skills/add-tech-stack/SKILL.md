---
name: add-tech-stack
description: Customize plugin stack skill for current project, or add new stack not shipped with plugin. Use when user types /add-tech-stack <stack> or wants to register/edit a stack skill.
---

This skill drives the `/add-tech-stack` slash command. It supports two scenarios:

- **Scenario A** — customize an existing registered stack (already in `.claude/stack-skills/registry.json` with `source: plugin`, `customized`, or `project`).
- **Scenario B** — create a brand-new stack not shipped with the plugin and not yet in the project registry.

It also accepts a `--force` flag (text-pattern detection in invocation; not real argv). When present, skip the file-choice prompt in Scenario A (force `[3] Both`) and show one confirmation about backups.

Authoritative spec: `docs/specs/2026-05-07-stack-customization-v5.5.0-design.md` (sections §5, §6, §7, §8). Snapshot model is described in §5.4 — project snapshots win over plugin defaults at runtime.

## a. Plugin Path Detection

The plugin is installed by different harnesses at different locations. Detect by trying these candidate dirs in priority order until one contains `docs/claude/templates/config.json`:

**Windows** (PowerShell, `$env:OS == "Windows_NT"`):
1. Claude Code marketplace: `$env:USERPROFILE\.claude\plugins\marketplaces\happypowerprocess\`
2. Codex (manual clone): `$env:USERPROFILE\.codex\happypowerprocess\`
3. Cursor extension: `$env:USERPROFILE\.cursor\extensions\happypowerprocess\`
4. OpenCode: `$env:USERPROFILE\.opencode\plugins\happypowerprocess\`

**macOS/Linux** (POSIX shell):
1. `$HOME/.claude/plugins/marketplaces/happypowerprocess/`
2. `$HOME/.codex/happypowerprocess/`
3. `$HOME/.cursor/extensions/happypowerprocess/`
4. `$HOME/.opencode/plugins/happypowerprocess/`

Store the first matching dir as `{PLUGIN_PATH}`. All template paths below are relative to `{PLUGIN_PATH}/docs/claude/templates/`.

**Fallback (if none of the candidates contain templates)**:
Run Glob across all 4 harness dirs in turn. Use the first match's grandparent-of-`docs/`-dir as `{PLUGIN_PATH}`:
- Windows: Glob in turn `$env:USERPROFILE\.claude\plugins\**\docs\claude\templates\config.json`, then `$env:USERPROFILE\.codex\**\docs\claude\templates\config.json`, then `.cursor\**`, then `.opencode\**`
- macOS/Linux: same pattern with `$HOME/.{claude,codex,cursor,opencode}/**/docs/claude/templates/config.json` (run 4 separate Globs since brace-expansion may not work in all shells)

If still no match → abort: `Plugin templates not found. Searched ~/.claude/, ~/.codex/, ~/.cursor/, ~/.opencode/. Verify happypowerprocess installed for your harness.`

## b. Argument Parsing

Per spec §7.0:

- If user runs `/add-tech-stack` with no stack argument → prompt: `❓ Stack name? (lowercase, hyphens; e.g., python-fastapi)`. Validate against regex `^[a-z][a-z0-9-]*$`. Re-prompt up to 3 times on validation failure. Abort after 3 with hint about allowed characters.
- If user runs `/add-tech-stack <name>` → use `<name>` directly (still validate against the same regex; same 3-retry rule).

**`--force` detection**: AI detects `--force` by case-insensitive text-pattern match in the user's invocation message. Look for tokens: `--force`, `force`. This is text-pattern detection, not real argv parsing (skills are markdown instructions, not executables). Store result as `{FORCE_MODE}` boolean.

Store the resolved stack name as `<name>` for use in subsequent sections.

## c. Reserved Names Check

Reject `<name>` if it equals one of these system-reserved names:

- `plugin`
- `customized`
- `project`
- `default`
- `template`

Output: `❓ '{name}' is reserved. Choose another.` No re-prompt; abort.

Note: the 5 plugin stack names (`dotnet`, `react`, `react-native`, `angular`, `iot-edge`) are **not** reserved here — they are present in the default registry and route to Scenario A (customize), not Scenario B.

## d. Registry Lookup → Scenario Branch

Read `.claude/stack-skills/registry.json` from project root.

If the file is missing, auto-create it from `{PLUGIN_PATH}/docs/claude/templates/registry.json`:
- Substitute `{PLUGIN_VERSION}` from `.claude-plugin/plugin.json` (`version` field).
- Substitute `{INIT_DATE}` = today's date in `YYYY-MM-DD`.
- Write the result to `.claude/stack-skills/registry.json` (create dir as needed).

Then branch:
- If `<name>` exists in `registry.stacks[]` (match by `name` field) → **Scenario A** (section e).
- If `<name>` does not exist → **Scenario B** (section f).

## e. Scenario A Flow (existing stack customize)

Per spec §7.1.

**Step e.1 — Detect existing customization.** If `.claude/stack-skills/<name>/SKILL.md` already exists (or `.claude/agents/implementer-<name>.md`), prompt:

```
📦 Stack '<name>' already has a project snapshot.
[continue overwrite / merge / cancel]
```

- `continue overwrite` → proceed to step e.2.
- `merge` → exit this skill and re-route the user to run `/sync-stack-skill <name>`.
- `cancel` → abort cleanly.

**Step e.2 — File choice prompt.** Display:

```
📦 Stack '<name>' found. ❓ Edit which file?
[1] SKILL.md only   [2] AGENT.md only   [3] Both   [4] Cancel
```

If `{FORCE_MODE}=true` → skip this prompt and force choice `[3] Both`.

**Step e.3 — Pre-flow backup.** If any current snapshot files exist, copy them to `.claude/stack-skills/<name>/backup-{YYYYMMDD-HHMMSS}/` (per spec §8.5 backup convention) before any write. Always backup `.claude/stack-skills/registry.json` and `CLAUDE.md` to the same backup dir.

**Step e.4 — Section walk.** For each target file (SKILL.md and/or AGENT.md per choice), walk every section. The plugin default content is read from:
- SKILL: resolve `skill_path` from registry entry → e.g. `{PLUGIN_PATH}/skills/implementer-dotnet-csharp/SKILL.md` (canonical agent/skill paths use full suffix like `implementer-dotnet-csharp` while user-facing stack name is `dotnet`).
- AGENT: resolve `agent_path` from registry entry → e.g. `{PLUGIN_PATH}/agents/implementer-dotnet-csharp.md`.

For each section, display the plugin default content and prompt: `[keep / override / append / skip]`.

- `keep` → copy plugin section verbatim into the snapshot.
- `override` → prompt user multi-line input (terminate with empty line); AI normalizes formatting; replace section in snapshot.
- `append` → prompt user multi-line input (terminate with empty line); AI appends below the plugin section content in snapshot.
- `skip` → AI inserts best-practice MINIMUM (slim values per spec §6.1 minimum-skip column for SKILL.md sections; per spec §6.2 minimum-skip column for AGENT.md sections).

**Step e.5 — Resolve canonical paths from current registry entry** (BEFORE any writes):
- `canonical_agent_basename` = basename of `entry.agent_path` without `.md` (e.g., `implementer-dotnet-csharp` from `agents/implementer-dotnet-csharp.md`).
- `snapshot_skill_path` = `.claude/stack-skills/<name>/SKILL.md` (where `<name>` is the registry's short-form name).
- `snapshot_agent_path` = `.claude/agents/<canonical_agent_basename>.md` (PRESERVES the canonical long-form so the snapshot truly overrides the plugin agent — runtime treats `implementer-dotnet-csharp` snapshot as overriding plugin's `implementer-dotnet-csharp` agent).

**Step e.6 — Write snapshot files** to the resolved paths:
- `snapshot_skill_path` (if SKILL chosen).
- `snapshot_agent_path` (if AGENT chosen).

**Step e.7 — Update `.claude/stack-skills/registry.json` for entry `<name>`:**
- `source` flips `plugin` → `customized` (or stays `customized` if already customized; stays `project` if already project).
- **`skill_path` MUST be rewritten to `snapshot_skill_path`** (e.g., `.claude/stack-skills/dotnet/SKILL.md`).
- **`agent_path` MUST be rewritten to `snapshot_agent_path`** (e.g., `.claude/agents/implementer-dotnet-csharp.md`).
- Without these rewrites, runtime lookup will not find the snapshot — load-bearing for F2/F3.

Validate registry per spec §5.6 BEFORE write (regex name, uniqueness, no `..` in paths).

**Step e.8 — Regenerate CLAUDE.md `<!-- stack-table -->` block.** If project root `CLAUDE.md` exists and contains both `<!-- stack-table:start ... -->` and `<!-- stack-table:end -->` markers, replace the entire content between markers from the registry (table format per spec §5.7). The Skill column now shows the `.claude/...` path for customized entries — that's correct.

If markers are missing, prompt: `❓ CLAUDE.md missing stack-table markers. Restore? [y/skip]`. Do **not** auto-fix — wait for user consent.

**Step e.9 — Auto-commit prompt.** Prompt: `❓ Auto-commit "feat: customize <name> stack skill"? [y/n]`.
- `y` → stage `.claude/stack-skills/`, `.claude/agents/`, `CLAUDE.md`; commit; if commit fails (pre-commit hook), show stderr and leave staged.
- `n` → skip with note: `Files written. Commit manually when ready.`

## f. Scenario B Flow (new stack create)

Per spec §7.2.

**Step f.1 — Confirmation.** Display:

```
⚠ Stack '<name>' not found in registry. Will create both SKILL.md + AGENT.md (no choice). Continue? [y/n]
```

- `n` → abort cleanly.
- `y` → continue.

**Step f.2 — Stack metadata (3 prompts).**

- `❓ Display label? (vd "Python / FastAPI"):` → store as `{STACK_LABEL}`.
- `❓ Description? (1 line):` → store as `{DESCRIPTION}`.
- `❓ Detection markers? (file globs, comma-separated; vd "pyproject.toml, main.py" or empty):` → store as `{DETECTION_MARKERS}` (array). Accept empty (warn: `⚠ Empty markers — brownfield auto-detect won't work for this stack`).

**Step f.3 — SKILL.md walk.** Read template from `{PLUGIN_PATH}/docs/claude/templates/stack-skill-template.md`.

For each fixed section (Required Rules / Quality Gates / Architecture / Anti-Patterns / Verification Matrix), prompt: `[input / suggest / skip]`.

- `input` → prompt user multi-line input (terminate with empty line); AI normalizes formatting and inserts.
- `suggest` → prompt: `❓ Use web search to enrich proposals? [y/n]`. Generate 2-3 best-practice approaches (use WebSearch only if user said `y`). User picks A/B/C or describes own. AI writes selected/described content.
- `skip` → AI inserts best-practice MINIMUM per spec §6.1 minimum-skip column for that section (e.g., Required Rules → 3 generic bullets; Architecture → "Use existing project structure. Match conventions in surrounding code.").

**Output Expectations** section: copy template default verbatim (no prompt — fixed by spec §6.1).

**Variable section loop.** After fixed sections, prompt: `❓ Add variable section? [add / done]`.
- `add` → prompt: `❓ Section name (markdown header without ##):`. Then prompt `[input / suggest / skip]` for that section's content. Repeat outer loop.
- `done` → exit loop.

**Step f.4 — AGENT.md walk.** Read template from `{PLUGIN_PATH}/docs/claude/templates/stack-agent-template.md`.

- `❓ Identity: who is this agent? (role + scope, e.g. "Python backend engineer specialized in FastAPI"):` → fills `{TASK_DOMAIN}` (infer from stack name + identity, e.g., "backend" / "frontend" / "mobile") and the identity sentence.
- `❓ Goal/focus (1 line):` → fills `{USER_GOAL}` and contributes to `{AGENT_DESCRIPTION}` (format: `"Use for implementing {STACK_LABEL} tasks with {USER_GOAL keywords}"`).
- Walk `## Execution rules` section with `[input / suggest / skip]` (same semantics as Step f.3) → fills `{EXECUTION_RULES}`.

Auto-fill remaining placeholders at write time:
- `{STACK_NAME}` = the resolved `<name>` from section b.
- `{STACK_LABEL}` = from metadata Q1 in step f.2.
- `{SKILL_PATH}` = `.claude/stack-skills/<name>/SKILL.md`.
- `{KEY_TOPICS}` = derived 3-4 topics from the just-written SKILL.md (read it back, list `## ` headers, pick the 3-4 most distinctive section names).

Output contract section: copy template default's 3 hardcoded bullets verbatim (no prompt — fixed by spec §6.2).

**Step f.5 — Resolve snapshot paths** (Scenario B has no plugin source, so canonical agent basename is `implementer-<name>`):
- `snapshot_skill_path` = `.claude/stack-skills/<name>/SKILL.md`
- `snapshot_agent_path` = `.claude/agents/implementer-<name>.md`

**Step f.6 — Write SKILL.md.** Write `.claude/stack-skills/<name>/SKILL.md` with full normalized content; all placeholders substituted. Create directory as needed.

**Step f.7 — Write AGENT.md.** Write `.claude/agents/implementer-<name>.md` (NOT a different short form — the canonical form for a new stack IS `implementer-<name>` since there's no plugin agent to mirror). Create directory as needed.

**Step f.8 — Append registry entry.** Append a new entry to `.claude/stack-skills/registry.json` with `source: "project"`, `created: {today_date_ISO}`, `skill_path: <snapshot_skill_path>`, `agent_path: <snapshot_agent_path>`:

```json
{
  "name": "<name>",
  "label": "{STACK_LABEL}",
  "description": "{DESCRIPTION}",
  "skill_path": ".claude/stack-skills/<name>/SKILL.md",
  "agent_path": ".claude/agents/implementer-<name>.md",
  "detection_markers": [...],
  "source": "project",
  "created": "{YYYY-MM-DD today}"
}
```

Validate per spec §5.6 BEFORE write (regex name, uniqueness, no `..` in paths). Backup current `registry.json` first.

**Step f.9 — Append to PROJECT.md.** Append `{STACK_LABEL}` (or `<name>` if PROJECT.md does not have a Stack section yet) to `.planning/PROJECT.md` Stack section. If `.planning/PROJECT.md` does not exist, skip silently with note: `ℹ .planning/PROJECT.md not found — skipped Stack section update.`

**Step f.10 — Regenerate CLAUDE.md `<!-- stack-table -->` block.** Same condition and behavior as Scenario A step e.8 (only if markers present; else prompt `❓ CLAUDE.md missing stack-table markers. Restore? [y/skip]`).

**Step f.11 — Auto-commit prompt.** Prompt: `❓ Auto-commit "feat: add <name> stack"? [y/n]`. Same behavior as Scenario A step e.9.

## g. Force Mode (`--force`)

When `{FORCE_MODE}=true`:

- **Scenario A**: skip the "edit which file?" prompt in step e.2 → force choice `[3] Both`. Show single confirmation up front:
  ```
  ⚠ Force mode: existing snapshots backed up to .claude/stack-skills/<name>/backup-{timestamp}/. Continue? [y/n]
  ```
- **Scenario B**: no behavior change (no file-choice prompt exists for Scenario B; both files are forced regardless).
- All other prompts (per-section `[keep/override/append/skip]` or `[input/suggest/skip]`, web-search opt-in, auto-commit) still run — `--force` is about skipping the file-choice prompt and confirming backup, not silencing every interaction.

## h. Rollback on Abort

Track every file written/modified during this skill invocation in a `creation_list`. On user Ctrl+C or AI write failure (per spec §8.5):

- Delete entries in `creation_list` that did not exist before invocation.
- Restore from `.claude/stack-skills/<name>/backup-{timestamp}/` (Scenario A) or revert append operations (Scenario B — partial `.claude/stack-skills/<name>/` directory deleted; PROJECT.md and CLAUDE.md appends reverted).
- Revert `.claude/stack-skills/registry.json` from backup (always backup registry before write).
- Revert `CLAUDE.md` from backup (always backup before regenerate).
- If cleanup itself fails → print partial-state summary: list of files left behind + advice to user (`Manual cleanup needed: <list>. Re-run with --force to overwrite.`).

## i. Validation Rules

Per spec §5.6 + §8.2:

- Stack name must match regex `^[a-z][a-z0-9-]*$`. Re-prompt up to 3 times on failure; abort after 3.
- Reserved names (system names only — see section c) → reject with no re-prompt.
- Path traversal: reject any path containing `..` (defensive — should not occur in normal flow but block if user input ends up shaping a path).
- Section content >10KB → warn but accept; AI normalizes.
- Empty `detection_markers` → allow with warning (legitimate for some stacks like `iot-edge`).
- Duplicate stack name in registry (Scenario B should never reach here because section d routes to Scenario A — but defensive check before write):
  ```
  Stack '{name}' already registered. Run /add-tech-stack {name} to edit.
  ```
  Abort.

## j. Web Search Policy (E3)

Per spec decision #5: web search is **always opt-in** via prompt. There is **no `--use-web` flag** anywhere in this skill.

The `suggest` action exists ONLY in Scenario B (and the T1 architecture detection sub-flow inside `/init-project`). When user picks `suggest`, prompt:

```
❓ Use web search to enrich proposals? [y/n]
```

- `y` → AI uses WebSearch to surface current best-practice references; combines with training knowledge; proposes 2-3 approaches.
- `n` → AI uses heuristic + training knowledge only; still proposes 2-3 approaches.

Scenario A actions (`keep / override / append / skip`) do NOT trigger web search.

E3 spirit: force the user to consent to token cost per use. Never call WebSearch without an explicit `y` to that prompt.

## AVOID

- Don't auto-derive AGENT.md from SKILL.md (decision #7 — user inputs separately).
- Don't auto-create snapshot in Scenario A — only after user confirms.
- Don't proceed past validation failures (3-retry abort rule).
- Don't skip rollback on write failure — partial state is unsafe.
- Don't write to plugin install directory — all writes go to `.claude/` and `.planning/` in project root.
- Don't add a `--use-web` flag — web search is always opt-in via prompt (E3 spirit).
- Don't auto-fix CLAUDE.md if markers are missing — prompt user first.

## Error Handling

| Scenario | Behavior |
|---|---|
| Plugin path / templates dir missing | Abort: `Plugin templates not found at {path}. Verify happypowerprocess installed: 'claude plugin list'` |
| Stack name regex fail | Re-prompt with hint, max 3 retries → abort |
| Reserved name | Reject with no re-prompt; abort |
| `registry.json` malformed JSON | Abort: `Registry malformed. Check .claude/stack-skills/registry.json or restore from registry.json.bak.` |
| Existing customization in Scenario A | Prompt `[continue overwrite / merge / cancel]` |
| Duplicate name in registry (defensive) | Show duplicate-name message; abort |
| User Ctrl+C mid-flow | Run rollback (section h); exit clean |
| AI write failure (disk, perms) | Stop, list partial files, run rollback |
| Auto-commit fails (pre-commit hook) | Show stderr, files remain staged, do not abort |
| CLAUDE.md missing entirely | Skip table regenerate with warning; suggest `/init-project` |
| CLAUDE.md markers missing | Prompt to restore; on `y` insert markers + content; on `skip` continue without table update |

**Constraints (do not violate):**

- Do NOT modify plugin install files — all snapshots go to project's `.claude/`.
- Do NOT auto-init git or auto-commit silently — always prompt.
- Do NOT call WebSearch without an explicit `y` to the per-section opt-in prompt.
- Do NOT regenerate CLAUDE.md table without confirming markers exist.
- Do NOT add fields to registry beyond spec §5.5 schema.
