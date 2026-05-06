---
name: update-config
description: Display current .planning/config.json and let user update specific fields with validation and mid-workflow impact warnings. Use when user types /update-config or says "update config", "change config", "change model", "change commit strategy".
---

This skill lets the user interactively edit `.planning/config.json` with field-level validation. Schema reference (single source of truth): `{PLUGIN_PATH}/docs/claude/config-schema.md`.

## a. Pre-check

In the target project root, verify `.planning/config.json` exists.
- Missing → abort: `Config not found. Run /init-project first.`
- Exists → continue.

## b. Plugin Path Detection

Detect plugin install path before reading the schema doc:
- Windows (PowerShell `$env:OS == "Windows_NT"`): `$env:USERPROFILE\.claude\plugins\marketplaces\happypowerprocess\`
- macOS/Linux (bash `[[ "$OSTYPE" == "msys" || "$OSTYPE" == "cygwin" ]]` false → POSIX): `$HOME/.claude/plugins/marketplaces/happypowerprocess/`

Store result as `{PLUGIN_PATH}`.

## c. Load Schema + Current Config

- Read `{PLUGIN_PATH}/docs/claude/config-schema.md` — AI reads the markdown table mentally for field names, valid values, and mid-workflow impact text. No programmatic parse needed.
- Read `.planning/config.json` — parse JSON. **Capture `mtime` of the file now** (used in section h to detect concurrent edits).
- Schema doc missing → abort: `Plugin schema doc malformed. Reinstall plugin.`
- Config malformed JSON → abort: `Config malformed JSON at line {N}. Suggest: restore from backup or run /init-project --force.`

## d. Display Current Config

Print a numbered table (10 entries; `commit_docs` splits into 2 rows, `model_defaults` groups under index 8 with 3 sub-fields):

```
Current config:
  1. mode:                              {value}
  2. granularity:                       {value}
  3. parallelization:                   {value}
  4. commit_docs.state_files:           {value}
  5. commit_docs.planning_artifacts:    {value}
  6. commit_atomic:                     {value}
  7. model_profile:                     {value}
  8. model_defaults.mechanical:         {value}
     model_defaults.standard:           {value}
     model_defaults.complex:            {value}
  9. workflow.research:                 {value}
 10. workflow.plan_check:               {value}
```

Index 8 covers all 3 `model_defaults` sub-fields together — when picked, walk through all 3 sequentially.

## e. User Picks Fields

- Prompt: `Which fields to change? (số, comma-separated, hoặc 'all')`
- Validate: comma-separated indices in range 1-10, or literal `all`.
- Invalid → re-prompt up to 3 times, then abort.
- Empty input or all "keep" → exit cleanly with `No changes made.`

## f. Per-Field Interactive Update Loop

For each picked index:
- Display: `[{field-name}] Current: {value}. Valid: {options from schema}`
- Prompt: `New value?`
- Validate against schema "Valid values" column.
- Invalid → re-prompt up to 3 times. After 3 failures → skip this field, continue to next (do **not** abort whole flow).
- Store accepted change in pending changes dict.
- Special case for index 8 (`model_defaults`): walk through `mechanical`, `standard`, `complex` in order; user can press Enter on any sub-field to keep current.

## g. Summary Diff

Show all pending changes:

```
Summary:
  granularity:    standard → coarse
  commit_atomic:  true → false
```

- Prompt: `Apply? [y/n]`
- `n` → exit, no changes written.
- `y` → continue to write.

## h. Write Config

- **Concurrent edit check**: re-stat `.planning/config.json` mtime. If different from the value captured in section c → abort: `Config modified externally during your edit. Re-run /update-config.`
- Write updated `.planning/config.json` (preserve original key ordering; pretty-print with 2-space indent).
- Write fails (permissions) → abort, original config unchanged.

## i. Side-effects

Only when specific fields changed:

- `commit_docs.planning_artifacts: true → false`:
  - Append the line `.planning/` to project's `.gitignore` (idempotent — skip if the uncommented line already present).
  - On write failure → warn: `Config updated but failed to update .gitignore. Add '.planning/' line manually.` (do not abort).
- `commit_docs.planning_artifacts: false → true`:
  - Remove the uncommented `.planning/` line from `.gitignore` if present. **Do not** touch the comment line `# .planning/`.
  - If line not present → silent no-op.

## j. Mid-Workflow Impact Warnings

For each changed field, display a warning using the **exact text** from the schema's "Mid-workflow impact" column:

```
⚠ granularity changed: Affects unplanned phases only. Already-planned phases unchanged.
⚠ commit_atomic changed: Apply from next task. Existing commits unchanged.
```

## k. STATE.md Audit Entry

Append to `.planning/STATE.md` "Notes" section:

```
- {YYYY-MM-DD HH:MM}: Config updated via /update-config (changed: {comma-separated field list})
```

If STATE.md has no "Notes" section → append the section header first, then the entry.

## l. Output Summary

Print:
- `✓ Updated .planning/config.json`
- Mid-workflow impact warnings (if any).
- Side-effects performed (e.g., `.gitignore` updated).
- One-line success: `Config update complete.`

## Error Handling

| Scenario | Behavior |
|---|---|
| `.planning/config.json` not found | Abort: `Config not found. Run /init-project first.` |
| Malformed JSON | Abort with line error + recovery suggestion |
| Schema doc missing | Abort: `Plugin schema doc malformed. Reinstall plugin.` |
| Invalid index input | Re-prompt valid range, max 3 retries → abort |
| Invalid value for field | Re-prompt with valid options, max 3 retries → skip field, continue to next |
| Write to `config.json` fails | Abort, original unchanged |
| `.gitignore` update fails | Warn but do not abort the whole flow |
| Concurrent edit (mtime mismatch) | Abort: `Config modified externally during your edit. Re-run /update-config.` |

**Constraints (do not violate):**
- Do NOT open an external editor (no `notepad`, `vim`, `code`).
- Do NOT auto-update without the section g apply prompt.
- Do NOT validate against custom rules — only the schema "Valid values" column.
- Do NOT modify other state files — only `.planning/config.json`, conditionally `.gitignore`, and the `.planning/STATE.md` audit entry.
- Do NOT add fields beyond what's in `config-schema.md`.
