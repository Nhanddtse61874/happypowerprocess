---
name: sync-stack-skill
description: Three-way merge a project stack snapshot with the latest plugin default. Use when user types /sync-stack-skill <stack> or wants to pull plugin updates into customized snapshot.
---

This skill drives the `/sync-stack-skill` slash command. It pulls the latest plugin default for a stack into the project's customized snapshot at `.claude/stack-skills/<name>/SKILL.md` (and optionally `.claude/agents/implementer-<name>.md`) using a per-section **3-way merge** (ancestor / plugin / snapshot).

It supports `--all` for batch sync over every registry entry whose `source != "project"`.

Authoritative spec: `docs/specs/2026-05-07-stack-customization-v5.5.0-design.md` — §7.3 (sync flow), §8.4 (conflict patterns), §8.3 (state mismatches), §5.4 (source state machine), §5.5 (registry schema), §12 (migration). Snapshot model lives in §5.4 — project snapshots win over plugin defaults at runtime.

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

- `/sync-stack-skill <stack>` → sync the single stack named `<stack>`. Validate `<stack>` against regex `^[a-z][a-z0-9-]*$` (per spec §5.6). Re-prompt up to 3 times on validation failure; abort after 3 with hint about allowed characters.
- `/sync-stack-skill --all` → loop over every registry entry where `source != "project"` (see section l).
- No argument → prompt: `❓ Stack name? (or 'all' to sync all):`. If user enters `all`, treat as `--all`. Otherwise validate per regex above.

Store the resolved stack name as `<name>` (single-stack flow) or set `{ALL_MODE}=true` (batch flow).

No `--use-web` flag is accepted — web search is not part of this skill (sync is a structural diff, not a content-generation flow).

## c. Pre-sync Checks

Read `.claude/stack-skills/registry.json` from project root. For each target `<name>` (single or per iteration of `--all` loop), apply these gates in order before any merge work:

1. **Entry not found in registry**:
   ```
   Stack '<name>' not in registry. Use /add-tech-stack <name> to register first.
   ```
   Abort (or skip + continue in `--all` loop).

2. **`entry.source == "project"`** (project-only stack — no plugin source):
   ```
   '<name>' is project-only (no plugin source). Sync not applicable. Use /add-tech-stack <name> to edit.
   ```
   Abort (or skip + continue in `--all` loop with reason logged).

3. **`entry.source == "plugin"` AND no `.claude/stack-skills/<name>/SKILL.md` exists**:
   ```
   '<name>' is using plugin default unchanged. Nothing to sync.
   ```
   Exit gracefully (or skip + continue in `--all` loop).

4. **`entry.source == "customized"` BUT snapshot file missing** (state mismatch per spec §8.3):
   - Warn: `⚠ Registry says 'customized' but snapshot file missing. Auto-fallback: resetting source to 'plugin'.`
   - Update registry: `source` = `plugin`. Validate per §5.6 before write. Save.
   - Exit (or continue in `--all` loop).

If all gates pass, proceed to section d.

## d. Common Ancestor Resolution

For a true 3-way diff we need the plugin version that the snapshot was originally synced from (the "ancestor"). Use `entry.synced_from_plugin_version` to identify the historical plugin file.

**v5.5.0 strategy** (no plugin git history available at runtime):

- Read plugin's CURRENT version from `{PLUGIN_PATH}/.claude-plugin/plugin.json` (`version` field). Store as `{PLUGIN_CURRENT_VERSION}`.
- If `entry.synced_from_plugin_version == {PLUGIN_CURRENT_VERSION}` → no plugin updates since last sync → output:
  ```
  No plugin updates since last sync (v{X}). Nothing to merge.
  ```
  Exit gracefully (or continue `--all` loop).
- If versions differ (`entry.synced_from_plugin_version != {PLUGIN_CURRENT_VERSION}`) but ancestor file is unavailable (we cannot retrieve historical plugin v{X} content from a registry without plugin git history) → fallback: treat plugin's CURRENT content as ancestor, performing a degraded **2-way diff** between snapshot and plugin v{Y}. Warn:
  ```
  ⚠ Cannot retrieve historical plugin v{X} content. Falling back to 2-way diff (snapshot vs plugin v{Y}). Conflict patterns may be approximate.
  ```

Future improvement (v5.6.0+): plugin keeps a per-version archive of its stack defaults so true 3-way diff is possible. Out of scope for v5.5.0.

## e. Section-by-Section Diff

Parse the snapshot at `.claude/stack-skills/<name>/SKILL.md` and the plugin default at `{PLUGIN_PATH}/<entry.skill_path>` (e.g., `{PLUGIN_PATH}/skills/implementer-dotnet-csharp/SKILL.md` — note canonical skill_path uses full suffix while user-facing stack name is shorter) into sections by markdown `## ` heading. Treat the frontmatter and any pre-`##` intro paragraph as a synthetic section "**Frontmatter+Intro**" merged together.

For each section, classify per spec §8.4:

| Pattern | Detection | Default action |
|---|---|---|
| Both unchanged | ancestor == plugin == snapshot | skip silently |
| Plugin updated, user unchanged | ancestor == snapshot, plugin != ancestor | take-plugin (no prompt) |
| User updated, plugin unchanged | ancestor == plugin, snapshot != ancestor | keep-mine (no prompt) |
| Both updated | ancestor != plugin AND ancestor != snapshot | per-section prompt |

For "Both updated" conflicts, prompt per section:

```
❓ Section ## <section name>: [keep-mine / take-plugin / manual-merge / show-detailed-diff]
```

- `keep-mine` → retain snapshot section content in merged output.
- `take-plugin` → replace snapshot section with plugin section content.
- `manual-merge` → enter the manual-merge sub-flow (section f).
- `show-detailed-diff` → display unified diff (ancestor → plugin and ancestor → snapshot) inline, then re-prompt the same 4-choice menu.

Record each per-section decision. Track an ordered list of merged section contents to assemble the final snapshot.

## f. Manual Merge Sub-flow

When user picks `manual-merge` for a "Both updated" section:

1. Display 3-pane view in transcript:
   ```
   ── ANCESTOR (plugin v{X}) ──
   [content]
   ── PLUGIN (v{Y}) ──
   [content]
   ── YOUR SNAPSHOT ──
   [content]
   ```
   (In degraded 2-way mode per section d, the ANCESTOR pane is omitted with a note.)

2. Prompt: `❓ Paste your manually-merged content (multi-line, end with empty line):`. Capture multi-line input until empty line terminator.

3. Prompt: `❓ Confirm save merged content? [y/edit/cancel]`.
   - `y` → record this content as the merged section; continue to next section.
   - `edit` → re-prompt the multi-line paste (step 2).
   - `cancel` → re-prompt the original 4-choice menu for this section (do not write yet — restoring pre-sync backup state if any partial writes already happened, see section j).

## g. Apply-All-Remaining Shortcut

After processing the FIRST conflict (Both-updated) section that the user resolves, offer:

```
❓ Apply same action to remaining non-conflict sections? [y/n]
```

- `y` → for the rest of the diff, auto-apply the same action class (`keep-mine` / `take-plugin`) to subsequent **non-conflict** sections (Plugin-updated / User-updated / Both-unchanged) that map cleanly to that action. Both-unchanged sections still skip silently regardless. **True conflicts (Both updated) ALWAYS prompt per-section** — they cannot be batched via this shortcut.
- `n` → continue per-section default behavior.

The shortcut is a one-shot offer per sync run — only emitted once after the first conflict resolution.

## h. Plugin-Removed Sections

If plugin's section X is gone but snapshot still has it:

```
❓ Plugin removed section "## X". Your snapshot has content for it. [keep / drop]
```

- `keep` → retain snapshot content in merged output (section stays in snapshot file).
- `drop` → omit section entirely from merged output.

## i. Plugin-Added Sections

If plugin has a section X that does not appear in snapshot or ancestor:
- Auto-add the plugin's section to the merged output (no prompt — additive operation; pure plugin contribution that does not conflict with anything user wrote).

## j. Pre-sync Backup

Before writing any merged snapshot content:
- Copy current `.claude/stack-skills/<name>/SKILL.md` → `.claude/stack-skills/<name>/SKILL.md.pre-sync-backup` (single file, **overwritten** on next sync per spec §5.5/§8.5 backup convention).
- If user re-syncs AGENT.md (section k step 3), do the same: `.claude/agents/implementer-<name>.md.pre-sync-backup`.

If user cancels mid-flow (Ctrl+C, or `cancel` repeatedly in manual-merge): restore from `.pre-sync-backup` and exit clean. Do not write merged content.

## k. Post-sync Updates

After all sections resolved and backup written:

1. **Write merged snapshot** to `.claude/stack-skills/<name>/SKILL.md`. Write the full assembled section list in original section order (plugin's section ordering wins for added sections; otherwise preserve current order).

2. **Update registry entry for `<name>`**:
   - `last_synced` = today (ISO date).
   - `synced_from_plugin_version` = current plugin version (`{PLUGIN_CURRENT_VERSION}` resolved in section d).
   - `source` stays `customized` (sync does NOT change source state — `add-tech-stack` is the only flow that flips source).
   - `skill_path` and `agent_path` REMAIN at `.claude/...` snapshot paths (sync does not move files — it merges content in place). If a project user previously edited registry to point back at plugin paths (manual revert), `/sync-stack-skill` will refuse with `'<name>' has no project snapshot at registered path; aborted. Use /add-tech-stack to re-customize.`
   - Validate per spec §5.6 BEFORE write (regex name, uniqueness, no `..` in paths).

3. **Re-sync AGENT.md prompt**:
   ```
   ❓ Re-sync AGENT.md too? [y/n]
   ```
   If `y`:
   - Snapshot agent path = `entry.agent_path` from registry (already `.claude/...` for `customized` source).
   - Plugin agent path = look up the plugin's original agent file by basename: take basename of `entry.agent_path` → search `{PLUGIN_PATH}/agents/<basename>.md`. (For `customized` entries, the registry's `agent_path` is the snapshot path; the plugin path is reconstructed from the basename pattern, since v5.5.0 doesn't store a separate `plugin_agent_path` field.)
   - Repeat sections d–j for the agent file pair. Same 4-pattern detection, same manual-merge sub-flow, same pre-sync backup convention.

   If `n` → skip.

4. **Regenerate CLAUDE.md `<!-- stack-table -->` block** — registry's `last_synced` changed, so the table's `Source` column or any displayed sync-state must reflect the new state. If project root `CLAUDE.md` exists and contains both `<!-- stack-table:start ... -->` and `<!-- stack-table:end -->` markers, replace the entire content between markers from the registry (table format per spec §5.7). If markers missing, prompt: `❓ CLAUDE.md missing stack-table markers. Restore? [y/skip]`. Do not auto-fix without consent.

5. **Auto-commit prompt**:
   ```
   ❓ Auto-commit "chore: sync <name> stack skill from plugin v{Y}"? [y/n]
   ```
   - `y` → stage `.claude/stack-skills/`, `.claude/agents/`, `CLAUDE.md`; commit; if commit fails (pre-commit hook), show stderr and leave staged.
   - `n` → skip with note: `Files written. Commit manually when ready.`

## l. `--all` Loop

For `/sync-stack-skill --all` (or no-arg flow that resolved to `all`):

- Iterate every registry entry where `source != "project"` (i.e., `source` in `{plugin, customized}`).
- For each entry, run sections c–k **except** the auto-commit prompt in step k.5 (defer to a single batch commit prompt at the end).
- On per-stack abort or skip (e.g., "nothing to sync", state mismatch auto-fallback), log the reason and **continue** with the next stack — do not abort the whole `--all` loop.
- Plugin stack removal detection (section m) runs per iteration.

After the loop:

- **Summary**:
  ```
  Synced: [list of stacks actually merged]
  Skipped: [list with reasons — e.g., "react: nothing to sync", "iot-edge: project-only"]
  Errors: [list with error messages, if any]
  ```

- **Single batch commit prompt**:
  ```
  ❓ Auto-commit "chore: sync all stack skills from plugin v{Y}"? [y/n]
  ```
  - `y` → stage all changed snapshot files + registry + CLAUDE.md; one commit.
  - `n` → skip with note: `Files written. Commit manually when ready.`

## m. Plugin Stack Removal Detection

During any sync (single or `--all`), if a registered stack has `source: customized` (or `source: plugin`) but `{PLUGIN_PATH}/<entry.skill_path>` does not exist (plugin removed it in a newer version):

```
⚠ Plugin no longer ships '<name>' (removed in v{Y}). Your snapshot still works. Convert source to 'project' to keep? [y/n]
```

- `y` → flip registry `source` to `project`. Remove the `synced_from_plugin_version` field (no longer applicable). Update `last_synced` to today. Save registry (validate per §5.6 first).
- `n` → leave as-is with warning logged in summary. The snapshot still works at runtime (project snapshot wins) but registry will continue to flag this on every future sync.

Do **not** auto-flip without user consent — always prompt.

## n. Validation

Same registry validation rules as `add-tech-stack` skill section i (per spec §5.6 + §8.2):

- Stack name regex `^[a-z][a-z0-9-]*$`.
- `name` unique within array.
- `source` in `{plugin, customized, project}`.
- Paths relative, no `..` (path traversal).
- Reserved names cannot be used (system-reserved: `plugin`, `customized`, `project`, `default`, `template`).
- Section content >10KB → warn but accept; AI normalizes.

Always re-validate registry BEFORE every write to `.claude/stack-skills/registry.json`. If validation fails → abort write, restore from backup, surface error to user.

## AVOID

- Don't apply changes silently — every conflict (Both updated) requires per-section user decision.
- Don't blindly trust apply-all on conflict sections — the shortcut covers only non-conflicts.
- Don't write merged content if user cancels manual-merge — restore pre-sync backup state and exit clean.
- Don't run sync on `source: project` stacks — they have no plugin source.
- Don't add `--use-web` flag (E3 conservative — no flag for this skill; sync is structural, not content generation).
- Don't auto-flip `source: customized` → `project` on plugin removal — always prompt user.
- Don't proceed if registry validation fails — abort, restore from backup, surface error.
- Don't modify plugin install files — all writes go to `.claude/` in project root.
- Don't skip the `.pre-sync-backup` write — it is the only rollback path.

## Error Handling

| Scenario | Behavior |
|---|---|
| Plugin path / templates dir missing | Abort: `Plugin templates not found at {path}. Verify happypowerprocess installed: 'claude plugin list'` |
| Stack name regex fail | Re-prompt with hint, max 3 retries → abort |
| `registry.json` malformed JSON | Abort: `Registry malformed. Check .claude/stack-skills/registry.json or restore from registry.json.bak.` |
| Stack not in registry | Output not-in-registry message; abort (single) or continue (`--all`) |
| `source: project` stack | Output project-only message; abort (single) or continue (`--all`) |
| Snapshot missing for `source: customized` | Warn + auto-fallback to `plugin`; exit (single) or continue (`--all`) |
| Plugin file missing for registered stack | Run plugin-removal detection (section m); prompt to convert to `project` |
| `synced_from_plugin_version == current` | Output nothing-to-merge; exit (single) or continue (`--all`) |
| Historical ancestor unavailable | Warn + fallback to 2-way diff per section d |
| User Ctrl+C mid-flow | Restore from `.pre-sync-backup`; revert registry from backup; exit clean |
| AI write failure (disk, perms) | Stop, list partial files, restore from `.pre-sync-backup`, restore registry backup |
| Auto-commit fails (pre-commit hook) | Show stderr, files remain staged, do not abort |
| CLAUDE.md missing entirely | Skip table regenerate with warning; suggest `/init-project` |
| CLAUDE.md markers missing | Prompt `❓ CLAUDE.md missing stack-table markers. Restore? [y/skip]` |

**Constraints (do not violate):**

- Do NOT modify plugin install files — all snapshots stay in project's `.claude/`.
- Do NOT auto-commit silently — always prompt.
- Do NOT regenerate CLAUDE.md table without confirming markers exist.
- Do NOT add fields to registry beyond spec §5.5 schema.
- Do NOT batch-apply across true conflicts — per-section prompts are mandatory.
