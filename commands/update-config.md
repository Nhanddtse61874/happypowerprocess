---
description: "Interactive update of .planning/config.json with field validation and mid-workflow impact warnings"
---

Invoke the **happypowerprocess:update-config** skill to update workflow config interactively.

The skill handles:
- Display current `.planning/config.json` as numbered table
- User picks fields to change (comma-separated indices or `all`)
- Per-field validation against `docs/claude/config-schema.md` (single source of truth)
- Summary diff before apply
- Auto-update `.gitignore` when `commit_docs.planning_artifacts` toggles
- Mid-workflow impact warnings per changed field
- STATE.md audit trail entry

**Trigger phrases** (also activates skill without the slash command):
- "update config", "change config", "change model", "change commit strategy"

If `.planning/config.json` doesn't exist, the skill aborts with a hint to run `/init-project` first.
