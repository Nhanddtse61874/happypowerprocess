---
name: init-project
description: Bootstrap project state files, config, CLAUDE.md, permissions, and gitignore for happypowerprocess workflow. Use when user types /init-project or wants to start a new project.
---

This skill bootstraps a new project for the happypowerprocess workflow. It runs **mechanically** — no vision brainstorming, no architecture mapping. Deep questioning is deferred to STEP 2 (`/brainstorm`).

Schema reference for all config fields: `{PLUGIN_PATH}/docs/claude/config-schema.md`.

Skill accepts a `--force` flag. When present, skip the idempotency menu, auto-select choice [4], show one confirmation: `⚠ Force mode: ALL existing state files will be backed up to .planning/backup-{timestamp}/ then overwritten. Continue? [y/n]`.

## a. Plugin Path Detection

Detect plugin install path before reading any template:
- Windows (PowerShell `$env:OS == "Windows_NT"`): `$env:USERPROFILE\.claude\plugins\marketplaces\happypowerprocess\`
- macOS/Linux (bash `[[ "$OSTYPE" == "msys" || "$OSTYPE" == "cygwin" ]]` false → POSIX): `$HOME/.claude/plugins/marketplaces/happypowerprocess/`

Store result as `{PLUGIN_PATH}`. All template paths below are relative to `{PLUGIN_PATH}/docs/claude/templates/`.

If `{PLUGIN_PATH}/docs/claude/templates/` is missing or empty → abort: `Plugin templates not found at {path}. Verify happypowerprocess installed: 'claude plugin list'`.

**Fallback (if hardcoded path missing)**:
If `{PLUGIN_PATH}/docs/claude/templates/config.json` does not exist, search broader:
- Windows: Glob `$env:USERPROFILE\.claude\plugins\**\docs\claude\templates\config.json` (max depth 4)
- macOS/Linux: Glob `$HOME/.claude/plugins/**/docs/claude/templates/config.json` (max depth 4)
- Use the first match's grandparent dir as `{PLUGIN_PATH}`
- If still no match → abort: `Plugin templates not found. Searched marketplace path and $HOME/.claude/plugins/. Verify happypowerprocess installed: claude plugin list`

## b. Detection Phase

In the target project root, check three things:
1. `.planning/STATE.md` exists? → if yes, trigger idempotency gate (section c).
2. `.git/` exists? → controls git init prompt (section g).
3. Brownfield markers? → use Glob for: `*.csproj`, `*.sln`, `package.json`, `Cargo.toml`, `pom.xml`, `pubspec.yaml`. If ANY match → brownfield mode. Store detected stacks in `{DETECTED_STACKS}`.

## c. Idempotency Gate

Runs only if `.planning/STATE.md` exists.

Display a status check (which of PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md, config.json, CLAUDE.md, .claude/settings.json exist). Then present:

```
[1] Abort (giữ nguyên)            ← DEFAULT
[2] Update config only             → invokes /update-config then exits
[3] Re-init missing files only     → create only missing, never overwrite
[4] Force re-init                  → backup ALL existing state files to .planning/backup-{YYYYMMDD-HHMMSS}/ then overwrite
```

Default = [1]. On `--force` → skip menu, jump to [4] with the single confirmation prompt described above.
  - **`--force` detection**: AI detects `--force` by text-pattern matching in the user's invocation message (case-insensitive). Look for tokens: `--force`, `force`, `force re-init`. This is text-pattern detection, not real argv parsing (skills are markdown instructions, not executables). If matched → skip the 4-choice menu above, go directly to choice [4] flow.

## d. Brownfield Handling

Runs only if `{DETECTED_STACKS}` is non-empty.
- Display: `📦 Brownfield detected: {DETECTED_STACKS}`.
- Skip stack prompt in section e — auto-fill `{STACK}` from `{DETECTED_STACKS}`.
- If multiple stacks detected and ambiguous (monorepo): prompt `Detected {N} stacks. Use all? [y/n] (else specify primary)` and resolve before proceeding.

## e. Interactive Prompts

Greenfield only (or override of brownfield auto-fill).

- **Q1**: `Project name?` → store as `{PROJECT_NAME}`. Validate regex `^[a-zA-Z0-9_\- ]+$`. Re-prompt up to 3 times on invalid input. After 3 failures → abort with hint about allowed characters.
- **Q2** (skip if brownfield): `Primary stack? (dotnet/react/react-native/angular/iot/mixed)` → store as `{STACK}`. Validate against listed values. Re-prompt up to 3 times. After 3 failures → abort.

`{INIT_DATE}` = today's date in `YYYY-MM-DD`.

## f. File Creation

Create in this order. For idempotency choice [3], skip any path that already exists. Track every file actually created in a creation list (used for rollback on error).

1. `mkdir .planning/`, `mkdir .planning/research/`, `mkdir docs/specs/` (no-op if present). Then create `.planning/research/.gitkeep` and `docs/specs/.gitkeep` (empty files — forces git to track these dirs; `.gitkeep` is a convention).
2. Copy templates from `{PLUGIN_PATH}/docs/claude/templates/` and fill placeholders:
   - `PROJECT.md` → `.planning/PROJECT.md` — fill `{PROJECT_NAME}`, `{STACK}`, `{INIT_DATE}` (Created field).
   - `REQUIREMENTS.md` → `.planning/REQUIREMENTS.md` — copy as-is.
   - `ROADMAP.md` → `.planning/ROADMAP.md` — copy as-is.
   - `STATE.md` → `.planning/STATE.md` — fill `Phase: STEP 0 — Bootstrap`, `Status: in_progress`, `Next Action: Run /brainstorm to start your first feature`, `Last updated: {INIT_DATE}`.
   - `config.json` → `.planning/config.json` — copy as-is (defaults already correct).
   - `CLAUDE.md` → `CLAUDE.md` (project root) — fill `{PROJECT_NAME}` and `{INIT_DATE}`.
   - `settings.json` → `.claude/settings.json` — copy as-is.
3. Create `.claude/settings.local.json` inline with content `{"permissions":{"allow":[]}}`.
4. Append `gitignore` template entries to project's `.gitignore`:
   - If `.gitignore` doesn't exist → create new with template content.
   - If exists → grep each entry; append only entries not already present (idempotent).
   - **Do NOT** add `.planning/` to gitignore — that line is owned by `/update-config` per `commit_docs.planning_artifacts` toggle (only added when set to `false`).

## g. Git Init Prompt

Runs only if no `.git/` was detected in section b.
- Display: `ℹ Folder chưa có git repo.`
- Prompt: `Init git? [y/n]`.
- `y` → run `git init`, confirm success.
- `n` → skip with note: `git init skipped. Run manually if needed.`
- If `.git/` exists but is broken (e.g., `git status` errors) → skip and warn: `Existing .git appears broken. Skipped git init.`

## h. Commit Prompt

Always runs after files are created.
- Display the list of files created.
- Prompt: `Auto-commit "init: project state files"? [y/n]`.
- `y`:
  - Read `.planning/config.json` → `commit_docs`.
  - If `commit_docs.state_files: true` → `git add` PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md, CLAUDE.md.
  - If `commit_docs.planning_artifacts: true` → also add `.planning/config.json` (and any other planning artifacts).
  - Always add `.claude/settings.json` and `.gitignore`.
  - `git commit -m "init: project state files"`.
  - If commit fails (pre-commit hook) → show stderr, leave files staged, do **not** abort.
- `n` → skip with note: `Files staged. Commit manually when ready.`

## i. STATE.md Audit Entry

Append to STATE.md "Notes" section:
- For greenfield: `- {YYYY-MM-DD HH:MM}: Project initialized via /init-project (Mode: greenfield, Stack: {STACK})`
- For brownfield: `- {YYYY-MM-DD HH:MM}: Project initialized via /init-project (Mode: brownfield, Stack: {STACK}) — Brownfield: run /brainstorm to map architecture`

## j. Output Summary

Print:
- List of files created.
- Git status (initialized? committed? commit hash if available?).
- Next-step guidance: `Run /brainstorm to start your first feature.`

## Error Handling

| Scenario | Behavior |
|---|---|
| Plugin path / templates dir missing or empty | Abort: `Plugin templates not found at {path}. Verify happypowerprocess installed: 'claude plugin list'` |
| Permission denied creating file | Abort + roll back files in creation list |
| User Ctrl+C mid-flow | Cleanup partial files in creation list, exit clean |
| Invalid project name (regex fail) | Re-prompt with hint, max 3 retries → abort |
| Invalid stack input | Re-prompt with valid options, max 3 retries → abort |
| `.git` exists but broken | Skip git init, warn: `Existing .git appears broken. Skipped git init.` |
| Auto-commit fails (pre-commit hook) | Show stderr, do not retry, files remain staged, do not abort |
| Disk full (ENOSPC) | Abort + cleanup partial files in creation list |
| Brownfield ambiguous (multiple monorepo markers) | Prompt user to confirm scope before proceeding |

**Constraints (do not violate):**
- Do NOT run vision/REQ-ID brainstorming — defer to STEP 2.
- Do NOT analyze code architecture — defer to brainstorm.
- Do NOT auto-init git silently — always prompt.
- Do NOT auto-commit silently — always prompt.
- Do NOT overwrite existing files in choice [3] — only create missing.
- Do NOT add features outside this spec.
