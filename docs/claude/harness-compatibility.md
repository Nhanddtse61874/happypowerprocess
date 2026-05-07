# Harness Compatibility Matrix

happypowerprocess workflow compatibility per AI harness. Active from v5.6.0+.

## Summary

| Workflow | Claude Code | Cursor | Codex | OpenCode |
|---|---|---|---|---|
| **Mode A (Solo)** | ✅ Full | ✅ Full | ✅ Full | ✅ Full |
| **Mode B (Team Spine)** | ✅ Full | ❌ Not supported | ❌ Not supported | ❌ Not supported |
| `/init-project` | ✅ slash command | ✅ slash command | ⚠ invoke skill by name | ✅ |
| `/update-config` | ✅ | ✅ | ⚠ skill by name | ✅ |
| `/add-tech-stack` | ✅ | ✅ | ⚠ skill by name | ✅ |
| `/sync-stack-skill` | ✅ | ✅ | ⚠ skill by name | ✅ |
| Hooks | ✅ `.claude/settings.json` | ⚠ `hooks-cursor.json` (different format) | ❌ no native hooks | ⚠ runtime-specific |
| Worktree parallelization | ✅ | ⚠ harness-dependent | ⚠ harness-dependent | ⚠ harness-dependent |

## Rationale: Why Mode B is Claude-only

Mode B requires three Claude-specific capabilities:

1. **Task tool with `subagent_type` + `model` + `isolation` parameters**
   - Claude Code: native support via Task tool
   - Other harnesses: agent APIs differ; cannot guarantee fresh isolated context per subagent dispatch

2. **Per-task model override using Claude model names** (`haiku` / `sonnet` / `opus`)
   - Claude Code: model parameter routes to correct Claude variant
   - Other harnesses: Codex uses GPT models, Cursor depends on user API config; literal Claude names don't transfer

3. **Worktree-isolated parallel subagent execution**
   - Claude Code: orchestrates parallel runs with git worktrees
   - Other harnesses: parallelization mechanism varies; may not isolate properly

Without all three, Mode B partial-fails: phase agents may load but dispatch behavior diverges, model selection silently degrades, parallel execution serializes or conflicts.

## Workaround for non-Claude harnesses

For complex tasks that would normally trigger Mode B on Claude:

- **Run Mode A solo.** All 11 workflow steps still apply.
- **Add manual review checkpoints.** Request review after each major step instead of via `qa-code-reviewer` subagent.
- **Do QA Gate manually.** Run `requesting-code-review` skill manually instead of dispatched.
- **Sequential execution.** No parallelization; one task at a time.
- **Document decisions inline.** Mode B's formal artifact templates can be filled manually if audit trail needed.

Outputs are slightly less formal but functionally equivalent for most projects.

## Detection mechanism

The Mode Selection Gate detects active harness via:

1. **Explicit user config** (highest priority): `.planning/config.json` field `active_harness`. Valid values: `auto`, `claude-code`, `cursor`, `codex`, `opencode`. Default `auto`.

2. **Environment variables** (when `auto`):
   - `$env:CLAUDECODE` (Windows) / `$CLAUDECODE` (POSIX) → `claude-code`
   - `$env:CURSOR_*` → `cursor`
   - `$env:CODEX_*` or `codex` in process → `codex`
   - `$env:OPENCODE_*` → `opencode`

3. **Fallback** (no env match): `claude-code`. This preserves current behavior for environments where detection is unclear; conservative default keeps Mode B available rather than silently disabling it.

## User override

Per the existing "user decision always wins" rule, a user CAN override Mode A force on non-Claude harness and pick Mode B anyway. The Mode Gate displays a warning before accepting:

```
⚠ You are forcing Mode B on <harness>. Subagent dispatch and model
selection may not work as designed. Proceed with manual review fallback ready.
```

Override is at user's risk. Common reasons to override: power user comfortable with manual fallback, debugging Mode B logic for cross-platform investigation.

## Future direction

v5.6.0 takes the simplest path: official Claude-only Mode B + runtime enforcement. Cross-platform Mode B emulation (e.g., model tier abstraction with per-platform mapping, harness adapter layer) is technically possible but requires:

- Build pipeline for per-platform bundles
- Coordinated release across 4 marketplaces
- Adapter abstractions for subagent dispatch + model selection
- Multi-month effort for solo maintainer

Deferred indefinitely. May reconsider at v7.0+ if project growth justifies investment.
