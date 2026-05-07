---
description: "Three-way merge a customized stack snapshot with the latest plugin default"
---

Invoke the **happypowerprocess:sync-stack-skill** skill to pull plugin updates into a project's customized stack snapshot.

The skill performs per-section 3-way merge between:
- **Common ancestor**: plugin version originally synced from (tracked in `.claude/stack-skills/registry.json` `synced_from_plugin_version`).
- **Plugin current**: latest plugin default at `skills/implementer-<stack>/SKILL.md`.
- **Your snapshot**: project customization at `.claude/stack-skills/<stack>/SKILL.md`.

**4 conflict patterns auto-classified:**
- Both unchanged → skip silently.
- Plugin updated, user unchanged → auto-take-plugin (no prompt).
- User updated, plugin unchanged → auto-keep-mine (no prompt).
- Both updated → per-section prompt `[keep-mine / take-plugin / manual-merge / show-detailed-diff]`.

**Manual-merge sub-flow**: 3-pane view (ancestor / plugin / snapshot) + paste merged content + confirm.

**Apply-all-remaining shortcut**: only available for non-conflict sections after the first conflict is resolved. True conflicts always prompt per-section.

**Pre-sync backup**: snapshot is copied to `.pre-sync-backup` (single file, overwritten on next sync) before merge writes.

**Plugin stack removal**: if plugin no longer ships a registered stack, prompts user to convert source to `project` (preserve snapshot) or leave as-is.

**`--all` flag**: loop sync over every registry entry where `source != "project"`. Single batch commit at end with summary (synced / skipped / errors).

Pass the user's full invocation message to the skill so it can detect arguments (stack name) and flags (`--all`).
