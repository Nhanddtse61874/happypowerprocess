# Config Schema (happypowerprocess)

This file is the single source of truth for all fields in `.planning/config.json`. It is consumed by the `/init-project` and `/update-config` skills. When this schema changes, both skills automatically pick up the new definitions — no other file needs to be updated.

---

## Schema Table

| Field | Type | Default | Valid values | Description | Mid-workflow impact |
|---|---|---|---|---|---|
| `active_harness` | enum | `auto` | `auto` / `claude-code` / `cursor` / `codex` / `opencode` | Active AI harness running this project. `auto` → detect via env vars, fallback `claude-code`. Used by Mode Selection Gate to enforce Mode B Claude-only policy (v5.6.0+). | Affects Mode Gate suggestion only; in-progress phases unaffected |
| `mode` | enum | `interactive` | `interactive` / `yolo` | Pause for confirmation each step vs auto-approve | Affects human touchpoints only, not completed steps |
| `granularity` | enum | `standard` | `coarse` / `standard` / `fine` | Phase count: 3-5 / 5-8 / 8-12+ | Affects unplanned phases only. Already-planned phases unchanged |
| `parallelization` | bool | `true` | `true` / `false` | Worktree-isolated parallel execution within waves | Apply from next wave |
| `commit_docs.state_files` | bool | `true` | `true` / `false` | Commit STATE.md, PROJECT.md, REQUIREMENTS.md, ROADMAP.md to git | Apply from next commit |
| `commit_docs.planning_artifacts` | bool | `true` | `true` / `false` | Commit `.planning/` files (research, plans, UAT, summaries). **`false` → auto add `.planning/` to `.gitignore`** | Apply from next commit + update `.gitignore` |
| `commit_atomic` | bool | `true` | `true` / `false` | `true` = commit per task (bisectable); `false` = batch commit per wave | Apply from next task. Existing commits unchanged |
| `model_profile` | enum | `balanced` | `balanced` / `quality` / `budget` | Shifts tier mapping: `budget` shifts all tiers down 1 (opus→sonnet, sonnet→haiku); `quality` shifts up 1 (haiku→sonnet, sonnet→opus). Clamped at boundaries | Unexecuted tasks use new model. Executed tasks unchanged |
| `model_defaults.mechanical` | string | `haiku` | model name (e.g., `haiku`, `sonnet`, `opus`) | Model for mechanical tasks (1-2 files, clear spec) | Unexecuted tasks use new model |
| `model_defaults.standard` | string | `sonnet` | model name | Model for integration tasks (multi-file, pattern matching) | Unexecuted tasks use new model |
| `model_defaults.complex` | string | `opus` | model name | Model for complex tasks (architecture, design judgment) | Unexecuted tasks use new model |
| `workflow.research` | bool | `true` | `true` / `false` | Enable research phase (STEP 4 of workflow) | Apply from next step |
| `workflow.plan_check` | bool | `true` | `true` / `false` | Enable Plan Checker (validates 11 dimensions, max 3 revision loops) | Apply from next plan |
| `workflow.memory_recall` | bool | `false` | `true` / `false` | Enable process-memory recall at STEP 0 (grep/glob top-K lessons before "keep or edit?") | Apply from next STEP 0 / resume |
| `workflow.telemetry` | bool | `false` | `true` / `false` | Enable per-task telemetry capture at STEP 7 and the SUMMARY telemetry table at STEP 11 | Apply from next task |
| `workflow.sandbox_verify` | bool | `false` | `true` / `false` | Enable bounded sandbox execute→observe→fix verification at STEP 8 (evidence-only, UAT preserved) | Apply from next STEP 8 |

---

## Display Grouping Convention

When `/update-config` displays the config table to the user, schema rows are grouped into top-level indices for ergonomic interaction. Group rules:

| Index | Schema rows covered | Notes |
|---|---|---|
| 0 | active_harness | Single field — project-context anchor (v5.6.0+) |
| 1 | mode | Single field |
| 2 | granularity | Single field |
| 3 | parallelization | Single field |
| 4 | commit_docs.state_files | Sub-field, displayed individually |
| 5 | commit_docs.planning_artifacts | Sub-field, displayed individually |
| 6 | commit_atomic | Single field |
| 7 | model_profile | Single field |
| 8 | model_defaults.mechanical + model_defaults.standard + model_defaults.complex | **Grouped**: picking index 8 walks all 3 sub-fields sequentially |
| 9 | workflow.research | Sub-field, displayed individually |
| 10 | workflow.plan_check | Sub-field, displayed individually |
| 11 | workflow.memory_recall | Sub-field, displayed individually |
| 12 | workflow.telemetry | Sub-field, displayed individually |
| 13 | workflow.sandbox_verify | Sub-field, displayed individually |

**Rule for new fields**: When adding a new schema row, decide if it should:
- (a) Get its own top-level index (default for independent fields)
- (b) Join an existing group (e.g., a new `model_defaults.review` tier joins index 8)

The `/update-config` skill derives the valid index range dynamically from this grouping — do NOT hardcode `0-10` (or any fixed range) in the skill.

---

## How to Use This Schema

### For `/init-project` skill

- Read the **Default** column to populate all fields when writing `.planning/config.json` for a new project.
- After writing defaults, apply `model_profile` shift to `model_defaults` if the profile is not `balanced`:
  - `budget`: shift all tiers down one (opus→sonnet, sonnet→haiku; haiku stays haiku)
  - `quality`: shift all tiers up one (haiku→sonnet, sonnet→opus; opus stays opus)

### For `/update-config` skill

- Load `.planning/config.json` and display all fields with numeric indices, showing current values.
- When the user selects a field to change, show the **Valid values** column as the prompt for acceptable input.
- Validate user input against the valid values list. Allow a maximum of 3 retries per field before skipping.
- After writing the updated config, display the **Mid-workflow impact** column as a warning for each changed field.

### Schema verification

- Before writing or displaying config, the skill should cross-reference the **Default** column in this table with the values in `docs/claude/templates/config.json`.
- If a mismatch is detected between this schema and the template file, abort immediately with: `Plugin schema drift. Reinstall plugin.`

---

## Permission Allowlist Note (Read-Only Trade-off)

The `/init-project` skill installs a default `.claude/settings.json` that pre-allows read-only operations: `Read`, `Glob`, `Grep`, and read-only `Bash` commands (`git status`, `ls`, `cat`, etc.).

**Implication**: The `Read` permission allows the AI to read ANY file the user has filesystem access to — including potentially sensitive files like `.env`, `~/.ssh/`, or credential stores. This is intentional for workflow ergonomics (the workflow is read-heavy), but users should be aware:

- ✅ **What is allowed without prompt**: Reading any file, listing/searching directories, checking git history
- ❌ **What still requires user approval**: All write/edit operations (`Edit`, `Write`, `NotebookEdit`), all destructive Bash commands (`rm`, `git commit`, `git push`, `npm install`), all stack-specific build commands

If this trade-off is unacceptable for a sensitive project, edit `.claude/settings.json` to remove `Read` from the allow list. The trade-off becomes: every file read triggers a permission prompt. For most workflow tasks (research, brainstorming, code review), this is acceptable.

This note exists in the schema doc (not the settings.json template itself) because JSON files cannot have comments.
