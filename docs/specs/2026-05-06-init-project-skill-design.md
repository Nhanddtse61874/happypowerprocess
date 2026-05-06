# Init-Project Skill — Design Spec

**Date:** 2026-05-06
**Status:** Approved (Section 1-5)
**Author:** Brainstorm session với user
**Implements:** New slash commands `/init-project` + `/update-config` cho happypowerprocess plugin

## Problem Statement

Plugin `happypowerprocess` hiện không có cơ chế tự động bootstrap project mới khi user cài đặt. User phải:
- Tạo thủ công các state files (PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md)
- Tạo `.planning/config.json` với 9 config fields
- Tạo CLAUDE.md project-level
- Setup `.claude/settings.json` permissions
- Setup `.gitignore` cho `.worktrees/` và optional `.planning/`
- Init git, first commit

→ User mới khó onboard, dễ miss files cần thiết, dễ mismatch với workflow expectations.

Đồng thời, sau khi init, user cần command tiện lợi để update config (mode, granularity, model_profile, commit strategy...) bất kỳ lúc nào — không phải edit JSON thủ công.

## Goals

1. **Skill `/init-project`**: bootstrap project mới (greenfield + brownfield) trong <30 giây với 2 prompts tối thiểu
2. **Skill `/update-config`**: interactive update config với validation + mid-workflow impact warnings
3. **Single source of truth** cho config schema (extensible khi thêm field mới)
4. **Idempotency**: re-run safe, có 4 lựa chọn xử lý khi project đã init
5. **Cross-platform**: Windows + macOS/Linux qua env vars

## Non-Goals

- Không brainstorm/deep questioning về vision (defer to STEP 2 brainstorming khi user start first feature)
- Không full architecture mapping cho brownfield (lightweight stack detection only)
- Không implement undo/rollback ngoài backup folder
- Không support multi-user concurrent edit
- Không migration từ old config schema (first version)

## Success Criteria

- User cài plugin → run `/init-project` → có working state trong <30 giây
- Config schema thay đổi → chỉ cần update 1 file (`config-schema.md`)
- Re-run `/init-project` không destroy work
- `/update-config` validate input, warn về mid-workflow impact, không break config

---

## Architecture

### File structure mới

```
{plugin-root}/
├── skills/
│   ├── init-project/
│   │   └── SKILL.md              ← MỚI (~120 dòng)
│   └── update-config/
│       └── SKILL.md              ← MỚI (~70 dòng)
└── docs/claude/
    ├── config-schema.md          ← MỚI (~80 dòng) — single source of truth
    ├── templates/
    │   ├── CLAUDE.md             ← Đã tạo
    │   ├── PROJECT.md            ← Đã có
    │   ├── REQUIREMENTS.md       ← Đã có
    │   ├── ROADMAP.md            ← Đã có
    │   ├── STATE.md              ← Đã có
    │   ├── config.json           ← Đã có
    │   ├── settings.json         ← MỚI (~15 dòng)
    │   └── gitignore             ← MỚI (~5 dòng)
    └── ...
```

### Component relationship

```
┌─────────────────┐     ┌─────────────────┐
│  /init-project  │     │ /update-config  │
│   SKILL.md      │     │   SKILL.md      │
└────────┬────────┘     └────────┬────────┘
         │   reads schema        │
         └───────┬───────────────┘
                 ▼
         ┌──────────────────┐
         │ config-schema.md │  ← Single source of truth
         └──────────────────┘
                 │ references
                 ▼
         ┌──────────────────┐
         │ templates/       │
         │ config.json      │  ← Physical defaults
         └──────────────────┘
```

### Plugin path detection (cross-platform)

Skills detect platform qua env vars:
- **Windows**: `%USERPROFILE%\.claude\plugins\marketplaces\happypowerprocess\`
- **macOS/Linux**: `$HOME/.claude/plugins/marketplaces/happypowerprocess/`

Detection logic trong skill: check `$env:OS == "Windows_NT"` (PowerShell) hoặc `[[ "$OSTYPE" == "msys" || "$OSTYPE" == "cygwin" ]]` (bash).

---

## Components

### 1. `docs/claude/config-schema.md` (single source of truth)

Bảng định nghĩa 9 fields với 6 cột:

| Field | Type | Default | Valid values | Description | Mid-workflow impact |
|---|---|---|---|---|---|
| `mode` | enum | `interactive` | `interactive` / `yolo` | Pause for confirm vs auto-approve | Affects human touchpoints only |
| `granularity` | enum | `standard` | `coarse` / `standard` / `fine` | Phase count: 3-5 / 5-8 / 8-12+ | Affects unplanned phases only |
| `parallelization` | bool | `true` | `true` / `false` | Worktree-isolated parallel execution | Apply from next wave |
| `commit_docs.state_files` | bool | `true` | `true` / `false` | Commit STATE.md, PROJECT.md... | Apply from next commit |
| `commit_docs.planning_artifacts` | bool | `true` | `true` / `false` | Commit `.planning/` files. **`false` → auto add to .gitignore** | Apply from next commit + update .gitignore |
| `commit_atomic` | bool | `true` | `true` / `false` | Commit per task vs batch per wave | Apply from next task |
| `model_profile` | enum | `balanced` | `balanced` / `quality` / `budget` | Shifts tier mapping. `budget` = down 1, `quality` = up 1 (clamped) | Unexecuted tasks dùng new model |
| `model_defaults` | object | `{mechanical:"haiku",standard:"sonnet",complex:"opus"}` | model name strings | Per-task model assignment | Unexecuted tasks dùng new model |
| `workflow.research` | bool | `true` | `true` / `false` | Enable research phase (Step 4) | Apply from next step |
| `workflow.plan_check` | bool | `true` | `true` / `false` | Enable Plan Checker (11 dimensions) | Apply from next plan |

Cộng thêm section "How to use this schema" cho 2 skills tham chiếu.

### 2. `skills/init-project/SKILL.md`

Frontmatter:
```yaml
---
name: init-project
description: Bootstrap project state files, config, CLAUDE.md, permissions, and gitignore for happypowerprocess workflow. Use when user types /init-project or wants to start a new project.
---
```

Body sections:
1. Detection phase — check `.planning/STATE.md`, `.git`, codebase markers
2. Idempotency gate — 4 choices nếu đã init
3. Brownfield stack detection — Glob markers
4. Interactive prompts — 2 questions: project name, primary stack
5. File creation — copy templates, fill placeholders ({PROJECT_NAME}, {INIT_DATE})
6. Git init prompt
7. Commit prompt
8. Output summary + next-step guidance

### 3. `skills/update-config/SKILL.md`

Frontmatter:
```yaml
---
name: update-config
description: Display current .planning/config.json and let user update specific fields with validation and mid-workflow impact warnings. Use when user types /update-config or says "update config", "change config", "change model", "change commit strategy".
---
```

Body sections:
1. Pre-check — verify `.planning/config.json` exists
2. Display current config với indices
3. User picks fields — comma-separated indices hoặc `all`
4. Per-field interactive update — read schema doc → validate input
5. Summary diff — show before/after, ask apply
6. Write config + warnings — mid-workflow impact per field
7. Side-effects — update `.gitignore` nếu `commit_docs.planning_artifacts` đổi

### 4. `docs/claude/templates/settings.json` (Option C — workflow-aware defaults)

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git status:*)",
      "Bash(git log:*)",
      "Bash(git diff:*)",
      "Bash(git branch:*)",
      "Bash(ls:*)",
      "Bash(cat:*)",
      "Bash(pwd)"
    ]
  }
}
```

Plus `.claude/settings.local.json`:
```json
{
  "permissions": {
    "allow": []
  }
}
```

### 5. `docs/claude/templates/gitignore`

```
# happypowerprocess workflow
.worktrees/
.claude/settings.local.json

# Conditional (added when commit_docs.planning_artifacts: false):
# .planning/
```

Skill **append** vào project's `.gitignore` (idempotent — skip if entry exists). If `.gitignore` không tồn tại → create new với template entries.

---

## Data Flow

### Init flow (Greenfield)

```
User: /init-project
│
├─ Detection: STATE.md exists? → NO → continue
├─ Detection: .git exists?
├─ Detection: Brownfield markers? → NO → greenfield mode
│
├─ Prompt 1: "Project name?" → {PROJECT_NAME}
├─ Prompt 2: "Primary stack?" → {STACK}
│
├─ Create files:
│   ├─ .planning/{PROJECT,REQUIREMENTS,ROADMAP,STATE}.md
│   ├─ .planning/config.json
│   ├─ .planning/research/
│   ├─ CLAUDE.md (with placeholders filled)
│   ├─ .claude/settings.json + settings.local.json
│   ├─ docs/specs/
│   └─ Append .gitignore entries
│
├─ Prompt 3 (if no .git): "Init git? [y/n]" → optional `git init`
├─ Prompt 4: "Auto-commit? [y/n]" → optional `git add` + `git commit`
│
└─ Output: Summary + next-step guidance
```

### Init flow (Brownfield)

Khác greenfield:
- Glob detect markers (`*.csproj`, `*.sln`, `package.json`, `Cargo.toml`, `pom.xml`, `pubspec.yaml`)
- Display: "📦 Brownfield detected: {stacks}"
- Prompt 2 SKIP (stack auto-filled, optional confirm)
- STATE.md adds note: "Brownfield project. Run /brainstorm to map architecture."

### Init flow (Already initialized — Idempotency Option B)

```
├─ STATE.md exists → display status check
├─ Prompt 4-choice menu:
│   [1] Abort (default)
│   [2] Update config only → invoke /update-config
│   [3] Re-init missing files only (skip existing)
│   [4] Force re-init (BACKUP existing → overwrite)
└─ Branch logic per choice
```

`--force` flag: skip menu, jump to choice 4 với 1 confirmation.

### Update-config flow

```
├─ Pre-check: config.json exists?
├─ Load schema + current config
├─ Display table với 10 numbered entries (commit_docs split into 2 lines, model_defaults split into 3)
├─ Prompt: "Which fields to change?" (comma-separated indices hoặc 'all')
├─ Per-field loop:
│   ├─ Display current + valid values
│   ├─ Prompt new value
│   └─ Validate (max 3 retries)
├─ Summary diff
├─ Prompt apply
├─ Write config
├─ Side-effects:
│   ├─ commit_docs.planning_artifacts: false → add `.planning/` to .gitignore
│   └─ commit_docs.planning_artifacts: true → remove `.planning/` from .gitignore
└─ Output: mid-workflow impact warnings per changed field
```

### Cross-skill interaction

- `/init-project` choice [2] invokes `/update-config` (chain)
- Both skills read same `config-schema.md` → consistent validation
- Both skills update STATE.md "Notes" section với audit trail

---

## Error Handling

### Init-project errors

| Scenario | Behavior |
|---|---|
| Plugin path không tồn tại | Abort: "Plugin templates not found at {path}" |
| Permission denied tạo file | Abort + roll back partial files |
| User Ctrl+C giữa flow | Cleanup partial files, exit clean |
| Project name có ký tự lạ | Re-prompt với hint (regex `[a-zA-Z0-9_\- ]+`) |
| Stack input không hợp lệ | Re-prompt với valid options |
| `.git` exists nhưng broken | Skip git init, warn |
| Auto-commit fails (pre-commit hook) | Show stderr, files staged, no abort |
| Disk full | Abort + cleanup partial files |
| Brownfield ambiguous (monorepo) | Display detected, ask user confirm |

### Update-config errors

| Scenario | Behavior |
|---|---|
| `.planning/config.json` không tồn tại | Abort: "Run /init-project first" |
| Malformed JSON | Abort + show line error, suggest restore |
| Invalid index | Re-prompt valid range |
| Invalid value (max 3 retries) | Skip current field, continue |
| Schema doc missing | Abort: "Plugin may be corrupted" |
| `.gitignore` update fails | Warn but don't abort |
| Concurrent edit detected (mtime) | Abort, instruct re-run |

### Backup behavior (force re-init)

- Backup dir: `.planning/backup-{YYYYMMDD-HHMMSS}/`
- Copy state files only (skip `.planning/research/`, `.planning/{phase}-*.md`)
- Display backup path before overwrite

### Validation pipeline

```
input → trim → match regex → check valid values → store
                  ↓ fail
             Re-prompt với hint (max 3 retries)
                  ↓ fail
             Abort current, return to previous step
```

### Audit trail (STATE.md "Notes")

```markdown
## Notes
- 2026-05-06 14:30: Project initialized via /init-project (Mode: greenfield, Stack: dotnet)
- 2026-05-06 15:45: Config updated via /update-config (changed: granularity, commit_atomic)
```

---

## Testing Approach

### Init-project test scenarios

| # | Scenario | Expected |
|---|---|---|
| T1 | Greenfield clean | All files created, prompts shown |
| T2 | Greenfield + .git existing | Skip git init prompt |
| T3 | Brownfield .NET | Stack auto = dotnet |
| T4 | Brownfield React | Stack auto = react |
| T5 | Brownfield mixed | Display 2 stacks, ask confirm |
| T6 | Re-run on initialized | 4-choice menu shown |
| T7 | Choice [3] missing files | Create missing only |
| T8 | Choice [4] force | Backup + overwrite |
| T9 | `--force` flag | Skip menu, jump to choice 4 |
| T10 | Invalid project name | Re-prompt với hint |
| T11 | Permission denied | Abort with clear error |
| T12 | Auto-commit fail | Show stderr, no abort |

### Update-config test scenarios

| # | Scenario | Expected |
|---|---|---|
| U1 | Single field change | Diff + apply + warning |
| U2 | Multiple fields | All prompted, applied |
| U3 | "all" | Walk all 9 fields |
| U4 | planning_artifacts true→false | `.planning/` added to .gitignore |
| U5 | planning_artifacts false→true | `.planning/` removed from .gitignore |
| U6 | Invalid value | Re-prompt valid options |
| U7 | No config exists | Abort with hint |
| U8 | Malformed JSON | Abort with line error |
| U9 | Pick 0 fields | Exit cleanly |
| U10 | STATE.md update | Notes entry added |

### Verification commands (Nyquist <60s)

Optional smoke test script `scripts/test-init-project.sh`:
```bash
[[ -f .planning/PROJECT.md ]] || exit 1
[[ -f .planning/config.json ]] || exit 1
cat .planning/config.json | python -m json.tool > /dev/null || exit 1
grep -q "{PROJECT_NAME}" CLAUDE.md && exit 1  # placeholders should be filled
```

### Manual test checklist

- [ ] All 12 init scenarios + 10 update scenarios pass
- [ ] CLAUDE.md placeholders rendered correctly
- [ ] `.gitignore` no duplicate entries on re-run
- [ ] Backup folder structure correct
- [ ] settings.local.json gitignored
- [ ] Cross-platform: Windows + macOS/Linux

### Schema validation

Skill = markdown instructions cho AI. AI reads `config-schema.md` (not programmatic parse) và verify khi load:
- All 9 fields có đầy đủ 6 cột (visual check)
- Defaults match `templates/config.json` (cross-reference)
- No duplicate field names (visual check)

Nếu schema thiếu / corrupt → AI abort: "Plugin schema doc malformed. Reinstall plugin."

---

## Design Decisions Log

| # | Decision | Rationale | Alternatives considered |
|---|---|---|---|
| 1 | Option C: mechanical + 2 prompts | Match user expectation, balance speed/depth | A (mechanical only), B (full deep questioning) |
| 2 | Option B: walk subset of fields | Speed when only changing 1-2 fields, validate per-field | A (walk all), C (open editor) |
| 3 | Option B: 4-choice menu on re-run | Cover all idempotency scenarios safely | A (hard abort), C (smart merge auto) |
| 4 | Option B: stack detection only | Avoid overlap with brainstorm; full architecture mapping is premature | A (greenfield always), C (full mapping) |
| 5 | A2+B3: ask both git init and commit | Explicit before side-effects; user mới hiểu rõ | A1+B1 (auto silent) |
| 6 | Option C: workflow-aware defaults | Smooth read-heavy ops, still safe for write/destructive | A (empty), B (minimal), D (per-stack) |
| 7 | Approach 2: shared schema doc | Single source of truth, future-proof | A1 (independent skills), A3 (single composite) |

---

## Out of Scope (Explicit)

| Item | Reason |
|---|---|
| Network template fetching | Templates local, no remote |
| User undo command | Backup folder sufficient |
| Encrypted config | No sensitive data in workflow config |
| Multi-user concurrent edit | Single-user repo assumed |
| Schema migration | First version, no legacy |
| Full brownfield architecture mapping | Belongs to STEP 2 brainstorming |
| Custom permission templates per stack | Out of scope, can extend later |

---

## Implementation References

- Plugin layout: see `.claude-plugin/plugin.json`
- Existing skill pattern: `skills/version/SKILL.md`
- Templates: `docs/claude/templates/`
- Workflow: `docs/claude/current-process-workflow.md`
- CLAUDE.md template: `docs/claude/templates/CLAUDE.md` (created in this work cycle)

---

## Last updated

2026-05-06
