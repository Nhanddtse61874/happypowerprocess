# Stack Customization (v5.5.0) — Design Spec

**Date:** 2026-05-07
**Status:** Draft (awaiting user spec review)
**Mode:** TBD (Mode Selection Gate runs after spec approval)
**Branch:** `feature/v5.5.0-stack-customization-2026-05-07`

---

## 1. Problem Statement

The plugin currently ships 5 stack skills (`dotnet`, `react`, `react-native`, `angular`, `iot-edge`) at `skills/implementer-*/SKILL.md`. These skills are **read-only at the plugin level** — users cannot:

1. **Customize per project**. Editing a stack skill modifies the plugin globally and affects every other project on the same machine.
2. **Add new stacks**. Projects using Python, Go, Rust, or any unsupported tech have no implementer skill/agent at all.
3. **Adapt to existing project architecture**. The current `.NET` skill says "Clean Architecture is a valid default, but do not force..." but provides no mechanism to record/follow the project's actual pattern. Users with Vertical Slices or N-Tier projects get Clean Architecture rules anyway because the rule is passive.

These constraints push users to either fork the plugin or hand-edit shared files — both fragile.

## 2. Goals (Functional)

- **F1 — Architecture detection (passive)**: provide an opt-in mechanism for AI to detect the project's actual architecture (folder structure + representative file content + opt-in web search) and write it into the project-scoped stack skill `## Architecture` section. Detection runs at user-initiated triggers only.
- **F2 — Customize stack skill per project**: allow users to override/append/keep individual sections of an existing plugin stack skill, scoped to the project.
- **F3 — Merge default + customized, scoped to project**: when user customizes, AI writes a full snapshot file that wins over the plugin default at runtime. Plugin defaults are never modified. Conflict policy: **user rules always win** over default.
- **F4 — `/add-tech-stack` slash command**: interactive, template-driven flow that walks the user through both `SKILL.md` and `AGENT.md` for new stacks (force both) and existing stacks (let user pick which file to edit).

## 3. Non-Goals

- Do **not** migrate the 5 plugin default skills to a new format — they continue to live at `skills/implementer-*/SKILL.md` unchanged.
- Do **not** modify runtime semantics of `stack-skill-rule-map.md` — the existing enforcement rule (every implementation task loads the relevant stack skill) stays the same.
- Do **not** support stack aliases (`dotnet` ≠ `csharp` ≠ `aspnet`) — each stack has one canonical name.
- Do **not** auto-trigger stack creation mid-task — when AI encounters an unregistered stack, it halts and asks the user to run `/add-tech-stack` (E3 conservative principle).
- Do **not** support cross-project stack sharing in v5.5.0 — copying `.claude/stack-skills/<stack>/` between projects is left as a manual operation.

## 4. Glossary

| Term | Meaning |
|---|---|
| **Plugin default** | Stack skill or agent file shipped with the plugin (read-only, lives under `<plugin-install>/skills/` and `<plugin-install>/agents/`). |
| **Project snapshot** | Full-content stack skill or agent file owned by a project at `.claude/stack-skills/<stack>/SKILL.md` or `.claude/agents/implementer-<stack>.md`. Wins over plugin default at runtime. |
| **Registry** | `.claude/stack-skills/registry.json` — the source of truth for stacks registered in a project (default 5 + user-added). |
| **Source state** | A registry entry's `source` field: `plugin` (using plugin default unchanged), `customized` (project snapshot derived from plugin default), `project` (project-only, no plugin source). |
| **Scenario A / Scenario B** | `/add-tech-stack` flow modes: A = customize an existing registered stack, B = create a brand-new stack. |
| **T1 / T2 / T3 / T4** | Architecture detection triggers: T1 = `/init-project` brownfield, T2 = unknown stack mid-task, T3 = `/add-tech-stack` Architecture section, T4 = `/sync-stack-skill`. |
| **Snapshot model** | Storage strategy where AI generates the full merged content at flow time and writes it to disk (vs. delta + runtime merge). |
| **Best-practice MINIMUM** | Slim placeholder content AI inserts when user explicitly skips a section — not full content, not empty. |

---

## 5. Architecture

### 5.1 Project-scoped file layout (after v5.5.0 usage)

```
<project-root>/
├── .claude/
│   ├── settings.json
│   ├── settings.local.json
│   ├── stack-skills/
│   │   ├── registry.json                    # source of truth
│   │   ├── dotnet/
│   │   │   └── SKILL.md                     # snapshot (only if customized)
│   │   ├── react/
│   │   │   └── SKILL.md
│   │   └── python-fastapi/
│   │       └── SKILL.md                     # user-created stack
│   └── agents/
│       ├── implementer-dotnet-csharp.md     # snapshot (only if customized)
│       ├── implementer-react-typescript.md
│       └── implementer-python-fastapi.md
├── .planning/
│   ├── PROJECT.md                           # Stack section enriched
│   ├── STATE.md
│   └── config.json
├── CLAUDE.md                                # has marker block <!-- stack-table:start/end -->
└── ...
```

### 5.2 Plugin-level layout (read-only)

```
<plugin-install-dir>/
├── skills/
│   ├── implementer-dotnet-csharp/SKILL.md   # plugin default — fallback only
│   ├── implementer-react-typescript/SKILL.md
│   ├── implementer-react-native-typescript/SKILL.md
│   ├── implementer-angular-typescript/SKILL.md
│   ├── implementer-iot-edge/SKILL.md
│   ├── add-tech-stack/SKILL.md              # NEW v5.5.0
│   ├── sync-stack-skill/SKILL.md            # NEW v5.5.0
│   └── init-project/SKILL.md                # updated — adds T1 sub-flow
├── agents/
│   ├── implementer-dotnet-csharp.md         # plugin default — fallback only
│   └── ... 4 more agents
├── commands/
│   ├── add-tech-stack.md                    # NEW
│   └── sync-stack-skill.md                  # NEW
└── docs/claude/templates/
    ├── registry.json                        # NEW — default 5 plugin stacks
    ├── stack-skill-template.md              # NEW
    ├── stack-agent-template.md              # NEW
    └── CLAUDE.md                            # updated — markers added
```

### 5.3 Runtime lookup order

When AI needs the skill or agent for stack `<X>`:

```
1. Read .claude/stack-skills/registry.json (source of truth).
2. Find entry where name == <X>:
   ├── Found, skill_path begins with .claude/...   → load project snapshot
   ├── Found, source == "plugin", path lives in plugin → load plugin default
   └── Not found                                    → halt, suggest /add-tech-stack <X> (T2)
```

Project snapshot **always** wins over plugin default if the file exists.

### 5.4 Source state machine

```
plugin       → /add-tech-stack <X> (Scenario A) → customized
customized   → /sync-stack-skill <X>            → customized (re-merged)
             → user deletes .claude/stack-skills/<X>/ → plugin (fallback)
project      → /add-tech-stack <X> (Scenario A) → project (edit)
             → /sync-stack-skill <X>             → project (no plugin source — sync only validates structure)
```

**Path rewrite on source transition (added in QA round 1):**

When `add-tech-stack` flips `source` `plugin → customized`, it MUST also rewrite the registry entry's `skill_path` and `agent_path` to point to the snapshot files:
- `skill_path` becomes `.claude/stack-skills/<name>/SKILL.md`.
- `agent_path` becomes `.claude/agents/<canonical_agent_basename>.md`, where `canonical_agent_basename` is the basename (without `.md`) of the *previous* `agent_path` value. This preserves the canonical long-form name (e.g., `implementer-dotnet-csharp.md`) so the snapshot truly overrides the plugin agent at runtime — they share the same agent name and the project snapshot wins.

Without this rewrite, runtime lookup (per §5.3) falls through to the plugin default and the customization has no effect. This is load-bearing for F2 and F3.

For Scenario B (new stack, `source: project`), the registry entry is created fresh at write time with paths already pointing to `.claude/...`. The canonical agent basename is `implementer-<name>` since there is no plugin source to mirror.

`/sync-stack-skill` does NOT change `source` or paths — it only merges content in place at the existing snapshot path.

### 5.5 Registry schema

```json
{
  "version": "1",
  "stacks": [
    {
      "name": "dotnet",
      "label": ".NET / C#",
      "description": "ASP.NET Core, EF Core, async backend",
      "skill_path": "skills/implementer-dotnet-csharp/SKILL.md",
      "agent_path": "agents/implementer-dotnet-csharp.md",
      "detection_markers": ["*.csproj", "*.sln"],
      "source": "plugin",
      "synced_from_plugin_version": "5.5.0",
      "last_synced": "2026-05-07"
    }
  ]
}
```

| Field | Required | Purpose |
|---|---|---|
| `name` | yes | Canonical lowercase identifier, regex `^[a-z][a-z0-9-]*$`. |
| `label` | yes | Display name shown in CLAUDE.md table. |
| `description` | yes | One-line summary. |
| `skill_path` | yes | Relative path to skill file. For `source: plugin`, points to plugin install dir. For `source: customized` or `project`, points to `.claude/stack-skills/<name>/SKILL.md` (rewritten by /add-tech-stack on source flip). |
| `agent_path` | yes | Relative path to agent file. For `source: plugin`, points to plugin install dir (canonical long-form name like `implementer-dotnet-csharp.md`). For `source: customized`, snapshot at `.claude/agents/<canonical_agent_basename>.md` (preserves long-form for true override). For `source: project`, snapshot at `.claude/agents/implementer-<name>.md`. |
| `detection_markers` | yes (can be `[]`) | File globs for brownfield detection + T2. |
| `source` | yes | `plugin` / `customized` / `project`. |
| `synced_from_plugin_version` | optional | Plugin version snapshot was synced from (sync command uses this). |
| `last_synced` | optional | ISO date of last sync. |
| `created` | optional | ISO date — present only when `source: project`. |

### 5.6 Validation rules

- `version` field exists, equal to `"1"`.
- `stacks` is an array.
- Each stack has all required fields.
- `name` matches regex `^[a-z][a-z0-9-]*$`.
- `name` unique within array.
- `source` in `{plugin, customized, project}`.
- Paths are relative, no `..` (path traversal).
- Reserved names cannot be used for new stacks: `plugin`, `customized`, `project`, `default`, `template`, plus the 5 plugin defaults.

### 5.7 CLAUDE.md marker block

`docs/claude/templates/CLAUDE.md` "Stack-Specific Rules" section uses marker comments:

```markdown
## Stack-Specific Rules

Khi gặp task code, AI **bắt buộc** load stack skill tương ứng. AI tự phân tích task → match đúng stack — không cần khai báo trước.

<!-- stack-table:start (managed by /add-tech-stack and /sync-stack-skill — do not edit manually) -->
| Stack | Label | Skill (Mode A) | Agent (Mode B) | Source |
|---|---|---|---|---|
| dotnet | .NET / C# | `skills/implementer-dotnet-csharp/SKILL.md` | `agents/implementer-dotnet-csharp.md` | plugin |
| react | React / TypeScript | `skills/implementer-react-typescript/SKILL.md` | `agents/implementer-react-typescript.md` | plugin |
| react-native | React Native / Expo | `skills/implementer-react-native-typescript/SKILL.md` | `agents/implementer-react-native-typescript.md` | plugin |
| angular | Angular / TypeScript | `skills/implementer-angular-typescript/SKILL.md` | `agents/implementer-angular-typescript.md` | plugin |
| iot-edge | IoT / MQTT / BLE | `skills/implementer-iot-edge/SKILL.md` | `agents/implementer-iot-edge.md` | plugin |
<!-- stack-table:end -->

Stack list expand được theo project — `/add-tech-stack` tự động regenerate table này từ `.claude/stack-skills/registry.json`.
```

`/add-tech-stack` and `/sync-stack-skill` replace the entire content between `<!-- stack-table:start ... -->` and `<!-- stack-table:end -->` from the registry.

---

## 6. Templates

### 6.1 `docs/claude/templates/stack-skill-template.md`

Used only for Scenario B (new stack creation) or `skip` minimum filling. Plugin defaults do not use this template.

```markdown
---
name: implementer-{STACK_NAME}
description: {DESCRIPTION}
---

# Implementer {STACK_LABEL}

Apply this skill for {STACK_LABEL} implementation tasks.

## Required Rules

{REQUIRED_RULES}

## Minimum Quality Gates

{QUALITY_GATES}

## Output Expectations

- Changed files with rationale
- Test and verification evidence
- Risk and follow-up items

---

## Architecture

{ARCHITECTURE}

---

{VARIABLE_SECTIONS}

---

## Anti-Patterns

{ANTI_PATTERNS}

---

## Verification Matrix

{VERIFICATION_MATRIX}
```

#### Placeholders and minimum-skip values

| Placeholder | Filled by | Best-practice MINIMUM (when explicit skip) |
|---|---|---|
| `{STACK_NAME}` | metadata Q (Step 1) | n/a (required) |
| `{STACK_LABEL}` | metadata Q | derived from `{STACK_NAME}` (title-case + spaces) |
| `{DESCRIPTION}` | metadata Q | "Implementation skill for {STACK_LABEL} tasks." |
| `{REQUIRED_RULES}` | input / suggest / skip | 3 generic bullets: `Keep contracts explicit. Validate at boundaries. Prefer simple over clever.` |
| `{QUALITY_GATES}` | input / suggest / skip | 3 generic bullets: `Add tests for changed behavior. Verify build passes. Note operational risks.` |
| `{ARCHITECTURE}` | input / suggest (with detection) / skip | "Use existing project structure. Match conventions in surrounding code." |
| `{VARIABLE_SECTIONS}` | user-driven add-loop | empty |
| `{ANTI_PATTERNS}` | input / suggest / skip | 3-row markdown table with placeholder rows ("_Add as discovered_"). |
| `{VERIFICATION_MATRIX}` | input / suggest / skip | 2-row markdown table with placeholder rows for Build / Tests. |

### 6.2 `docs/claude/templates/stack-agent-template.md`

```markdown
---
name: implementer-{STACK_NAME}
description: {AGENT_DESCRIPTION}
model: inherit
---

You implement {TASK_DOMAIN} tasks in {STACK_LABEL}. {USER_GOAL}

**REQUIRED:** Read and apply `{SKILL_PATH}` before writing any code. That skill is the authoritative guide for {KEY_TOPICS}.

Execution rules:
{EXECUTION_RULES}

Output contract:
- Changed files and rationale
- Tests added/updated and results
- Risk notes and operational considerations
```

| Placeholder | Filled by | Skip minimum |
|---|---|---|
| `{STACK_NAME}` | from registry | n/a |
| `{AGENT_DESCRIPTION}` | derived: `Use for implementing {STACK_LABEL} tasks with {USER_GOAL keywords}` | "Use for implementing {STACK_LABEL} tasks." |
| `{TASK_DOMAIN}` | identity Q | inferred from `{STACK_NAME}` (e.g., "backend", "frontend", "mobile") |
| `{STACK_LABEL}` | from registry | n/a |
| `{USER_GOAL}` | goal Q | empty string |
| `{SKILL_PATH}` | auto from registry `skill_path` | n/a |
| `{KEY_TOPICS}` | derived: 3-4 section names from SKILL.md | "the patterns and verification matrix in that skill" |
| `{EXECUTION_RULES}` | input / suggest / skip | "- Match existing project conventions.\n- Add tests for changed behavior.\n- Note operational risks." |

### 6.3 `docs/claude/templates/registry.json` (default plugin entries)

`/init-project` copies this to `.claude/stack-skills/registry.json` after substituting `{PLUGIN_VERSION}` and `{INIT_DATE}`.

```json
{
  "version": "1",
  "stacks": [
    {
      "name": "dotnet",
      "label": ".NET / C#",
      "description": "ASP.NET Core, EF Core, async backend",
      "skill_path": "skills/implementer-dotnet-csharp/SKILL.md",
      "agent_path": "agents/implementer-dotnet-csharp.md",
      "detection_markers": ["*.csproj", "*.sln"],
      "source": "plugin",
      "synced_from_plugin_version": "{PLUGIN_VERSION}",
      "last_synced": "{INIT_DATE}"
    },
    {
      "name": "react",
      "label": "React / TypeScript",
      "description": "React web with hooks, TanStack Query, Zustand",
      "skill_path": "skills/implementer-react-typescript/SKILL.md",
      "agent_path": "agents/implementer-react-typescript.md",
      "detection_markers": ["package.json"],
      "source": "plugin",
      "synced_from_plugin_version": "{PLUGIN_VERSION}",
      "last_synced": "{INIT_DATE}"
    },
    {
      "name": "react-native",
      "label": "React Native / Expo",
      "description": "Mobile with React Navigation, MMKV, TanStack Query",
      "skill_path": "skills/implementer-react-native-typescript/SKILL.md",
      "agent_path": "agents/implementer-react-native-typescript.md",
      "detection_markers": ["app.json", "metro.config.js"],
      "source": "plugin",
      "synced_from_plugin_version": "{PLUGIN_VERSION}",
      "last_synced": "{INIT_DATE}"
    },
    {
      "name": "angular",
      "label": "Angular / TypeScript",
      "description": "Angular 17+ with Signals, RxJS, NgModule or standalone",
      "skill_path": "skills/implementer-angular-typescript/SKILL.md",
      "agent_path": "agents/implementer-angular-typescript.md",
      "detection_markers": ["angular.json"],
      "source": "plugin",
      "synced_from_plugin_version": "{PLUGIN_VERSION}",
      "last_synced": "{INIT_DATE}"
    },
    {
      "name": "iot-edge",
      "label": "IoT / MQTT / BLE",
      "description": "Edge device connectivity with MQTT/BLE/firmware",
      "skill_path": "skills/implementer-iot-edge/SKILL.md",
      "agent_path": "agents/implementer-iot-edge.md",
      "detection_markers": [],
      "source": "plugin",
      "synced_from_plugin_version": "{PLUGIN_VERSION}",
      "last_synced": "{INIT_DATE}"
    }
  ]
}
```

---

## 7. Command Flows

### 7.0 No-argument invocation

If user runs `/add-tech-stack` without a stack argument, AI prompts: `❓ Stack name? (lowercase, hyphens; e.g., python-fastapi)`. Validates against the regex from §5.6. Re-prompts up to 3 times on validation failure, then aborts.

After name resolves, the flow branches into Scenario A or B based on whether the name exists in the registry.

### 7.1 `/add-tech-stack <stack>` — Scenario A (existing stack customize)

Trigger: stack is in registry with `source: plugin` (or `customized`).

```
[User]: /add-tech-stack dotnet

AI: 📦 Stack 'dotnet' found. ❓ Edit which file?
    [1] SKILL.md only   [2] AGENT.md only   [3] Both   [4] Cancel
```

For each section in target file, AI shows the plugin default content and asks: `[keep / override / append / skip]`.

- `keep` → copy plugin section verbatim
- `override` → user pastes new content (free-form draft); AI normalizes format and replaces section
- `append` → user pastes additional rules; AI appends below plugin section
- `skip` → AI inserts best-practice MINIMUM (slim) for that section

After both files are walked, AI:
1. Writes `.claude/stack-skills/<stack>/SKILL.md` and/or `.claude/agents/implementer-<stack>.md`.
2. Updates `registry.json`: `source` flips to `customized` (if was `plugin`) and stays `project` (if was `project`).
3. Regenerates the CLAUDE.md `<!-- stack-table -->` block.
4. Asks `❓ Auto-commit "feat: customize <stack> stack skill"? [y/n]`.

### 7.2 `/add-tech-stack <stack>` — Scenario B (new stack create)

Trigger: stack is **not** in registry.

Forces both SKILL.md and AGENT.md (no choice prompt).

```
[User]: /add-tech-stack python-fastapi
AI: ⚠ Stack 'python-fastapi' not found. Will create both SKILL.md + AGENT.md. Continue? [y/n]
```

Three steps:

**Step 1 — Stack metadata** (3 prompts): `label`, `description`, `detection_markers`.

**Step 2 — SKILL.md** (walk template sections; menu per section: `[input / suggest / skip]`):
- `input` → user pastes content; AI normalizes and writes
- `suggest` → AI proposes 2-3 best-practice approaches (with optional web search per E3); user picks; AI writes
- `skip` → AI fills with best-practice MINIMUM
- After 6 fixed sections, AI enters a "variable section" loop: `❓ Add variable section? [add / done]`. User can add `## State Management`, `## Async I/O Patterns`, etc. as desired.

**Step 3 — AGENT.md** (3 user touchpoints):

- **Q-identity**: "Bạn (agent) là ai?" → fills `{TASK_DOMAIN}` and identity line.
- **Q-goal**: "Focus chính?" → fills `{USER_GOAL}` (one-liner appended to identity sentence) and `{AGENT_DESCRIPTION}`.
- **Walk `## Execution rules` section** with `[input / suggest / skip]` — fills `{EXECUTION_RULES}`.

The remaining placeholders are auto-filled at generation time: `{STACK_NAME}` and `{STACK_LABEL}` from registry, `{SKILL_PATH}` from registry, `{KEY_TOPICS}` derived from the just-written SKILL.md section headers, and the Output contract is the template's 3-bullet default (no user input). AGENT.md does **not** track SKILL.md changes — `{KEY_TOPICS}` is captured at generation time only.

After both files are walked, AI:
1. Writes both files.
2. Adds entry to `registry.json` with `source: project` and `created: {date}`.
3. Appends label to `.planning/PROJECT.md` Stack section.
4. Regenerates CLAUDE.md marker block.
5. Asks `❓ Auto-commit "feat: add <stack> stack"? [y/n]`.

### 7.3 `/sync-stack-skill <stack>` (or `--all`) — 3-way merge

Trigger: user wants to pull plugin updates into a customized snapshot.

Computes a 3-way diff per section:

| Pattern | Detection | Default action |
|---|---|---|
| Both unchanged | ancestor == plugin == snapshot | skip silently |
| Plugin updated, user unchanged | ancestor == snapshot, plugin diff | take-plugin (no prompt) |
| User updated, plugin unchanged | ancestor == plugin, snapshot diff | keep-mine (no prompt) |
| Both updated | ancestor != plugin AND ancestor != snapshot | per-section prompt |

Per-section prompt: `[keep-mine / take-plugin / manual-merge / show-detailed-diff]`.

`apply-all-remaining` shortcut is offered **only for non-conflict** sections. True conflicts always require per-section decision.

Manual-merge sub-flow displays a 3-pane view (ancestor / plugin / snapshot) and asks user to paste the merged content.

After all sections resolved, AI updates the snapshot file, sets `last_synced` and `synced_from_plugin_version` in registry, optionally re-syncs AGENT.md the same way, regenerates CLAUDE.md table, and asks for commit confirmation.

### 7.4 `/init-project` enhancement (T1 sub-flow)

After existing brownfield detection finds stack markers, the new sub-flow runs (E3 conservative — every step asks consent):

```
AI: ❓ Scan architecture for detected stack(s) too? [y/n]
[y]
AI: For 'dotnet': ❓ Use web search to verify against current best practice? [y/n]
[n]
AI: Scanning folder structure (Glob)... reading 2-3 representative files...
AI: Detected pattern: Clean Architecture (4-layer) — confidence HIGH
AI: ❓ Save this to .claude/stack-skills/dotnet/SKILL.md ## Architecture? [y / edit / skip]
[y]
AI: Creating .claude/stack-skills/dotnet/ snapshot from plugin default + ## Architecture replaced.
AI: Registry source 'plugin' → 'customized'.
```

Greenfield (no markers): T1 sub-flow is skipped entirely.

### 7.5 T2 — Unknown stack mid-task

Trigger: AI receives a task that mentions a stack not in registry.

```
[User task]: "Add a Go endpoint for /orders using Fiber"

AI: ⚠ Stack 'go-fiber' not registered in this project's registry.json.
AI: I cannot proceed without a stack skill.
AI: 💡 Suggestions:
    [1] Run /add-tech-stack go-fiber  (interactive)
    [2] Cancel task
❓ Pick [1/2]:
```

E3 conservative: **no auto-create option**. User must explicitly invoke `/add-tech-stack`.

T2 detection heuristic: AI scans the task text for known framework keywords (FastAPI, Fiber, Axum, Spring, etc.) and for file globs from registered `detection_markers`. False positives: AI prefers to ask rather than over-trigger.

---

## 8. Error Handling and Edge Cases

### 8.1 Concurrent edit / file conflict

| Case | Action |
|---|---|
| `/add-tech-stack <X>` with existing customization | Prompt: `[continue overwrite / merge / cancel]`. `merge` re-routes to `/sync-stack-skill <X>`. |
| `registry.json` corrupt | Abort with restore hint. Search for `registry.json.bak` or suggest `/init-project --force`. |
| Two concurrent sessions editing registry | v5.5.0 does **not** handle file locks — single-user repo assumed. Document in skill: "Don't run `/add-tech-stack` in two terminals simultaneously." |

### 8.2 Validation failures

| Case | Action |
|---|---|
| Stack name regex fail | Re-prompt up to 3 times with hint. Abort after 3. |
| Reserved name | Reject with no re-prompt. |
| Duplicate name in registry | Show: stack exists, suggest `/add-tech-stack` (Scenario A) or `/sync-stack-skill`. |
| Empty `detection_markers` | Allow (legitimate for `iot-edge`). Warn: brownfield auto-detect won't work for this stack. |
| Section content >10KB | Warn but accept; AI normalizes. |

### 8.3 Plugin / project state mismatches

| Case | Action |
|---|---|
| Registry has `source: plugin` but plugin file missing | Runtime warn. User runs `/sync-stack-skill <X>` or `/add-tech-stack <X> --force`. |
| Registry has `source: customized` but snapshot file missing | Auto-fallback to plugin default; reset registry source → `plugin`; warn. |
| Registry has `source: project` but snapshot file missing | Hard error at load. User recreates via `/add-tech-stack` or removes registry entry. |
| Plugin upgrade adds new stack | `/sync-stack-skill --all` prompts: "Plugin introduces new stacks: [...]. Add to project? [y/n]". |
| Plugin upgrade removes a stack | `/sync-stack-skill` warns: "Plugin no longer ships '<X>'. Convert source to 'project' to keep? [y/n]". |

### 8.4 Sync conflict resolution

The 4-pattern detection table from §7.3 applies. Manual-merge sub-flow lets user paste the final content; AI confirms before save.

### 8.5 Failure recovery / rollback

| Failure point | Rollback |
|---|---|
| Ctrl+C mid Scenario A | Restore `.claude/stack-skills/<stack>/backup-{timestamp}/`; revert registry; revert CLAUDE.md. |
| Ctrl+C mid Scenario B | Delete partial `.claude/stack-skills/<stack>/`; revert registry append; revert PROJECT.md and CLAUDE.md appends. |
| AI write fail (disk, perms) | Stop, list partial files, suggest manual cleanup + retry with `--force`. |
| Sync mid-flow fail | Restore from `.claude/stack-skills/<stack>/SKILL.md.pre-sync-backup`. |

Backup convention:
- Pre-flow: `.claude/stack-skills/<stack>/backup-{YYYYMMDD-HHMMSS}/`
- Pre-sync: `<filename>.pre-sync-backup` (single, overwritten on next sync)
- No automatic cleanup — user manages or defers to a future `/cleanup-stack-backups` command (out of scope for v5.5.0).

### 8.6 Detection edge cases (F1)

| Case | Action |
|---|---|
| 0 patterns matched | "No architecture markers found. Skip detection? [y/manual-input]". |
| Conflicting patterns | "Multiple patterns detected: [...]. Pick primary or describe own:". |
| Web search fail | Fallback to heuristic-only with warning. |
| Confidence < 50% | Show + warn; always prompt user explicitly. |

### 8.7 CLAUDE.md marker block edge cases

| Case | Action |
|---|---|
| Markers missing | Prompt to restore; on `y`, re-insert markers and content. |
| Manual edits between markers | Prompt: `[overwrite / preserve-manual / cancel]`. `preserve-manual` skips CLAUDE.md update only. |
| CLAUDE.md missing entirely | Skip with warning; suggest `/init-project`. |
| Markers inside fenced code block | Abort regenerate; surface as user error. |

---

## 9. Testing Strategy

Plugin self-dev does not run a unit test framework — testing is manual eval + adversarial pressure (consistent with v5.4.0).

### 9.1 Eval scenarios (must pass before release)

| ID | Scenario | Pass criteria |
|---|---|---|
| E1 | Greenfield init + `/add-tech-stack python-fastapi` (Scenario B) | Registry, both files, CLAUDE.md table, PROJECT.md, commit all updated correctly. |
| E2 | Brownfield `.NET` init + T1 detection | Heuristic detects Clean Architecture, `## Architecture` section written, source flips to `customized`. |
| E3 | `/add-tech-stack dotnet` (Scenario A) — test all 4 actions | `keep` copies verbatim, `override` replaces, `append` appends below default, `skip` inserts minimum. |
| E4 | `/sync-stack-skill dotnet` after plugin v5.6.0 | Auto-take/auto-keep behaviors and per-section prompts work; manual-merge sub-flow works. |
| E5 | T2 — task mentions `go-fiber` | AI halts; offers only 2 options ([1] `/add-tech-stack`, [2] cancel). No auto-create option. |
| E6 | Re-run `/add-tech-stack` on customized stack | Detects existing customization; offers `[continue / merge / cancel]`. |
| E7 | Ctrl+C mid Scenario B | No partial files; registry, PROJECT.md, CLAUDE.md unchanged. |
| E8 | Validation failures | Uppercase, reserved, traversal, duplicate all rejected with appropriate messages. |

### 9.2 Adversarial pressure (5 sessions per skill)

1. Standard happy path
2. Vague answers ("dunno", "whatever")
3. Contradictions ("override" then "keep default")
4. Attacks (path traversal, shell injection in inputs)
5. Long-running (10+ stacks added in one session)

### 9.3 Verification commands (QA Gate)

```bash
# Schema integrity
test -f docs/claude/templates/registry.json && jq empty docs/claude/templates/registry.json
test -f docs/claude/templates/stack-skill-template.md
test -f docs/claude/templates/stack-agent-template.md

# Skill files exist
test -f skills/add-tech-stack/SKILL.md
test -f skills/sync-stack-skill/SKILL.md
test -f commands/add-tech-stack.md
test -f commands/sync-stack-skill.md

# CLAUDE.md template has markers
grep -q "stack-table:start" docs/claude/templates/CLAUDE.md
grep -q "stack-table:end" docs/claude/templates/CLAUDE.md

# Plugin manifest
jq '.version' .claude-plugin/plugin.json   # expect "5.5.0"
jq '.keywords' .claude-plugin/plugin.json | grep -E "(add-tech-stack|stack-customization|sync-stack-skill)"

# stack-skill-rule-map.md references registry
grep -q "\.claude/stack-skills/registry.json" docs/claude/stack-skill-rule-map.md
```

### 9.4 Out of scope for v5.5.0 testing

- Multi-platform plugin path detection (assumed working from v5.4.0).
- Performance / quantitative SLAs.
- File lock / concurrency testing (single-user assumed).
- Network failure modes for web search beyond basic fallback.
- Long-term backup retention.

### 9.5 Release acceptance criteria

- All 8 eval scenarios pass.
- Adversarial pressure (5 sessions) passes for both new skills.
- All verification commands return success.
- CHANGELOG entry written.
- Version bump applied to all 5 plugin manifests.
- Plugin keywords updated (add `add-tech-stack`, `stack-customization`, `sync-stack-skill`).
- No regressions in v5.4.0 commands (`/init-project`, `/update-config`).

---

## 10. Decisions Log

| # | Decision | Rationale |
|---|---|---|
| 1 | Section-level merge (not full-replace, not rule-level diff) | Stack skills are rich; full-replace loses default value, rule-level diff over-engineered. Sections balance flexibility and simplicity. |
| 2 | Snapshot model (Hybrid C: snapshot + `/sync-stack-skill`) | Predictable mental model — user opens file and sees full effective rules. Drift handled by explicit re-sync, not runtime parser. |
| 3 | User rules win on conflict | User explicitly customizes for project-specific reason; default should never override deliberate customization. |
| 4 | E3 Conservative for architecture detection | Zero surprise; user fully controls scan/web/save decisions; reproducible. Trade-off: more prompts per init (~3/stack). |
| 5 | Web search always opt-in (no `--use-web` flag) | E3 spirit; force user to consent to token cost per use; flag would be hard to remember and undermine consent. |
| 6 | Template SKILL.md = 3 fixed top + Architecture + N variable + 2 fixed bottom | Matches structure of all 5 existing plugin skills; consistent ordering for AI normalization. |
| 7 | AGENT.md user inputs separately (not auto-derived from SKILL) | User-correctable identity prompts; AI still keeps content concise (template stays under 25 lines). When SKILL changes, AGENT does **not** auto-regenerate — user re-runs `/add-tech-stack` if they want new agent rules. |
| 8 | Force both SKILL+AGENT on Scenario B; ask file choice on Scenario A | Scenario B is creation — both files needed up front. Scenario A is editing — user often only wants to tweak one. No `--skill-only` flag (poor UX, hard to remember). |
| 9 | Best-practice MINIMUM on explicit skip (Q4.3 option d) | User skip is an action, not silence. Slim minimum signals "intentionally light here" without empty section confusion. |
| 10 | Single registry.json (Q5.a option A) | Adequate for v5.5.0 scale; per-stack meta files would fragment without proportional benefit. Migration to per-stack meta is possible at v6.0. |
| 11 | CLAUDE.md marker block + auto-regenerate (Q5.b option A) | Pattern already used (`<!-- gitnexus:end -->`); inline table keeps AI context fresh; auto-regenerate keeps drift impossible. |
| 12 | Reserved names enforced (`plugin`, `customized`, `project`, `default`, `template`, plus 5 plugin defaults) | Prevent ambiguity in registry semantics and avoid accidental override of plugin defaults via Scenario B. |
| 13 | T2 unknown stack: halt + 2 options, no auto-create | E3 principle: AI never silently scaffolds. User must explicitly invoke `/add-tech-stack`. |
| 14 | Pre-flow backup directory + pre-sync `.pre-sync-backup` file | Two backup conventions cover the two failure shapes: customize abort vs sync abort. No auto-cleanup — user controls retention. |
| 15 | No file lock for concurrent editing | Single-user repo assumed; locks add code without solving the actual usage pattern. Document in skill instead. |

---

## 11. Open Questions

None at spec stage. Mode Selection Gate runs after user spec review and may surface additional implementation-level questions (e.g., subagent dispatch strategy, model assignment per task) which will be answered in the writing-plans step.

---

## 12. Appendix

### 12.1 Why "happypowerprocess" branding stays

The personalized fork of Superpowers (this repo) brands itself as `happypowerprocess`. Existing stack agent names (`implementer-dotnet-csharp`, etc.) and skill paths use the unbranded form — v5.5.0 keeps that convention. New stacks get the same convention: `implementer-<stack>`.

### 12.2 Relationship to v5.4.0 init-project skill

`/add-tech-stack` re-uses several patterns from `/init-project`:

- Plugin path detection (`{PLUGIN_PATH}` resolution + Glob fallback).
- 2-prompt minimal interactive style.
- `--force` flag for power users (skips confirmations, backups still created).
- Idempotency menu for re-runs on existing customization.
- Template-driven file creation with rollback list on abort.
- Auto-commit prompt at the end.

`/init-project` itself is updated to add the T1 sub-flow (architecture detection on brownfield) as a new section between current sections `d` (Brownfield Handling) and `e` (Interactive Prompts).

### 12.3 Migration path for users on v5.4.0

User upgrades plugin to v5.5.0:
1. Existing projects continue to work — no `.claude/stack-skills/` directory yet, AI uses plugin defaults exactly as before.
2. First time user runs `/add-tech-stack` or `/sync-stack-skill`, the registry is auto-created if missing (read template, fill `{PLUGIN_VERSION}` = current, `{INIT_DATE}` = today).
3. Existing CLAUDE.md without markers: AI prompts user to add markers (or recommends `/init-project --force` to refresh template).

No breaking changes for v5.4.0 users who don't invoke v5.5.0 commands.
