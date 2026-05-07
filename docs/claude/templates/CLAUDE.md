# {PROJECT_NAME} — Project Guidelines for Claude

## Workflow Mandate

Project này sử dụng **happypowerprocess** workflow. Mọi task (feature, bugfix, refactor, hotfix) **bắt buộc** đi qua 11-step workflow định nghĩa dưới đây.

Mode (A solo / B team spine) **không fix cứng** ở project level — được quyết định ở **STEP 3 (Mode Selection Gate)** cho từng task dựa trên 5 scoring criteria.

**Note (v5.6.0+):** Mode B is Claude Code-only. On Cursor / Codex / OpenCode, Mode Gate auto-forces Mode A regardless of complexity score (per `harness-compatibility.md` in plugin docs dir).

## Plugin References

Plugin install dir (auto-detected by platform):
- **Windows**: `%USERPROFILE%\.claude\plugins\marketplaces\happypowerprocess\docs\claude\`
- **macOS/Linux**: `$HOME/.claude/plugins/marketplaces/happypowerprocess/docs/claude/`

Tài liệu chính (relative path từ plugin docs dir):
- `current-process-workflow.md` — 11-step unified workflow (primary reference)
- `state-files-guide.md` — state files guide (PROJECT, STATE, REQUIREMENTS, ROADMAP)
- `templates/` — templates for state and phase output files
- `research-phase-guide.md` — research agents and outputs
- `mode-selection-criteria.md` — Mode A vs B scoring (chạy ở STEP 3)
- `master-dispatcher-prompt.md` — task routing
- `agent-output-templates.md` — output template contracts (Mode B)
- `runtime-modes.md` — Mode A and B details
- `stack-skill-rule-map.md` — mandatory stack skills
- `full-ai-team-setup.md` — AI team topology (Mode B)

## 11-Step Workflow (Authoritative)

- **STEP 0 — Resume / New project**:
  - **Resume**: read `.planning/STATE.md` + `.planning/config.json` → display config → ask "keep or edit?" → update if needed → continue.
  - **New project**: collect config upfront → save `.planning/config.json` (mode, granularity, parallelization, commit_docs, commit_atomic, model_profile, model_defaults, workflow flags) → deep questioning → create state files from plugin templates.
  - `commit_docs`: `{state_files: bool, planning_artifacts: bool}` — when `planning_artifacts: false`, `.planning/` auto-added to `.gitignore`.
  - `commit_atomic`: `true` (per task — bisectable) / `false` (batch per wave).
  - `model_profile`: `balanced` / `quality` / `budget` — shifts tier mapping (budget → all down one, quality → all up one, clamped).
  - `model_defaults`: `{mechanical: "haiku", standard: "sonnet", complex: "opus"}` — per-task model assignment.
  - **Artifact persistence**: write to disk immediately, never hold in memory.

- **Config Update (anytime)**: User says "update config" / "change config" / "change model" / "change commit strategy" at any time → AI reads `.planning/config.json`, displays it, lets user edit, updates immediately. If changes affect completed steps → warn + suggest re-plan.

- **STEP 1 — Fast Lane Check** (`fast-lane-assessment-v1`): run before every task. If eligible (5 criteria all true: scope clear, ≤2 files, no arch/API change, no security/migration, low regression) → skip Steps 2-4.

- **STEP 2 — Brainstorm** (`skills/brainstorming/SKILL.md`): required unless Fast Lane. Output: approved design direction + `.planning/REQUIREMENTS.md` with REQ-IDs (testable, user-centric, atomic). Every v1 requirement must map to exactly one phase — 100% coverage required.

- **STEP 3 — Mode Selection Gate**: score 5 criteria (domain count, risk, QA gate, cross-team, output format), suggest mode (A or B), wait for user approval. Mode locked after approval — source of truth for downstream phases.

- **STEP 4 — Research**: skip if Fast Lane or `workflow.research: false`. Each agent loads CONTEXT.md + `.planning/config.json` + this CLAUDE.md before researching. Mode A: 2 agents (Stack + Pitfall). Mode B: 4 agents (Stack + Feature + Architecture + Pitfall) + Research Synthesizer. **All claims must have `[VERIFIED/CITED/ASSUMED]` tags** — never present assumed knowledge as fact.

- **STEP 5 — Spec**: Mode A → brainstorming skill solo. Mode B → `phase-discovery-lead` + `phase-architecture-lead` (input: brainstorm + research, no re-brainstorm).

- **STEP 6 — Plan**: goal-backward methodology (Observable Truths → Required Artifacts → Required Wiring → Key Links → `must_haves` frontmatter). XML tasks with `<model>`, `<read_first>`, `<action>`, `<verify>` (Nyquist Rule: automated <60s), `<done>`. **`<model>` required per task** (`haiku`/`sonnet`/`opus`) — planner assigns from `model_defaults`, user overrides before approval. **Context budget**: ~50% per plan, max 2-3 tasks. Wave assignment algorithm: dependencies + file overlap → force later wave; same-wave plans must have **zero file overlap**. **Plan Checker** validates 11 dimensions (max 3 revision loops) if `workflow.plan_check: true`.

- **STEP 7 — Execute (Wave by Wave)**:
  - **Intra-wave file overlap check** before execution — 2 plans modify same file → force sequential.
  - **Pre-task confirmation** (interactive mode): show model + files, ask if user wants to switch model.
  - Controller dispatches subagent with model from `<model>` tag (or user override).
  - **Escalation**: haiku → sonnet → opus if subagent BLOCKED.
  - **Commit strategy confirmation** before first wave: confirm `commit_atomic` (true = commit per task, false = batch per wave).
  - **Worktree isolation** when `parallelization: true` — sequential dispatch (avoids `.git/config.lock` race condition), parallel run, merge worktree branches back to main after wave.
  - Commit behavior per `commit_docs` + `commit_atomic` config.
  - **Stack skill mandatory** — match task domain → load skill from `stack-skill-rule-map.md`.
  - **Failure recovery**: retry / skip / abort per plan; partial progress recorded in STATE.md for resume.

- **STEP 8 — UAT + Verification**:
  - **8a UAT**: User tests, AI does **not** claim done. AI creates `.planning/{phase}-UAT.md`, shows expected behavior, asks user to confirm or describe differences. AI infers severity from user description.
  - **8b Goal-Backward Verification**: AI cross-references `must_haves` (truths/artifacts/key_links) with implementation artifacts → `.planning/{phase}-VERIFICATION.md`.
  - Gap closure if needed (back to Step 7 with fix plans).
  - **Regression gate + schema drift detection** (Mode B): run prior phase tests + verify TS build + DB schema sync.

- **STEP 9 — QA Gate**: Mode A → `requesting-code-review` skill. Mode B → `phase-qa-gate` + `qa-code-reviewer`. Severity: Critical / Important / Suggestion. Block → fix → Step 7. Approve / Approve with conditions → Step 10.

- **STEP 10 — Release/DevOps**: Mode A → `finishing-a-development-branch` skill. Mode B → `phase-release-devops-lead` + `devops-cicd-assistant`. Skip in Mode A if no formal release gate needed.

- **STEP 11 — Ship**: PR/merge + `.planning/{phase}-SUMMARY.md` + update `.planning/ROADMAP.md` + update `.planning/STATE.md`. Escalation if unresolved risk/conflict (present 2-3 options for user to decide).

- **Never auto-advance**: Stop after each step, wait for user confirmation.

- **Full reference**: see `current-process-workflow.md` in plugin docs dir for complete workflow detail.

## Code-Level Behavioral Guidelines

The 11-step workflow above defines WHAT/WHEN. This section defines HOW when actually touching code in STEP 7.

### Think Before Coding
- When code relies on an unverified assumption, tag `[ASSUMED]` in commit message or state file (extends the `[VERIFIED/CITED/ASSUMED]` mechanism from STEP 4 to code-level).
- If a REQ-ID has multiple interpretations, surface all of them — do not silently pick one.
- If a simpler approach than the approved plan exists, push back with justification before coding.

### Simplicity First
- Write the minimum code that satisfies the REQ-ID. Nothing speculative.
- No abstractions, flexibility, or configurability that were not requested.
- No error handling for impossible scenarios.
- Self-check: "Would a senior engineer call this overcomplicated?"
- **Conflict resolution**: If this conflicts with Plan Checker's 11 dimensions, prefer simplicity and flag it in QA Gate (STEP 9).

### Surgical Changes
- Only modify what is strictly required for the current task/REQ-ID.
- Do not "improve" surrounding code, formatting, or comments.
- Do not refactor things that are not broken; match existing style.
- Unrelated dead code → mention it in the PR description, do not delete.
- **Cleanup boundary**: Only remove imports/variables/functions that your own changes made orphan.
- Every changed line must trace back to a REQ-ID or a task in the plan.

### Goal-Driven Execution
Already covered by STEP 6 (goal-backward + `must_haves` frontmatter + `<verify>` tag) and STEP 8 (Goal-Backward Verification). Not repeated here.

## Branching Rule

- **Prefer feature branches for non-trivial work.**
- Branch naming format: `feature/<feature-name>-yyyy-mm-dd`. Example: `feature/auth-system-{INIT_DATE}`.
- Sync latest `main` before creating a new feature branch.
- If user explicitly asks to commit/push directly to `main`, follow the user request.
- **Never skip git hooks** (`--no-verify`) without explicit user request.

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

Stack list expand được theo project — `/add-tech-stack` tự động regenerate table này từ `.claude/stack-skills/registry.json`. Reference chính: `stack-skill-rule-map.md` trong plugin docs dir.

## Project-Specific Deviations from Plugin Defaults

Ghi lại mọi quyết định lệch chuẩn so với plugin workflow (vd: dùng format khác, skip step, override config).

| Deviation | Rationale | Date |
|-----------|-----------|------|
<!-- Example row (xóa comment khi điền entry thật):
| Plan X uses Markdown checkbox instead of XML contract | Already authored before deviation discovered | YYYY-MM-DD |
| `workflow.research: false` for Phase 0 | Foundation phase has no domain research need | YYYY-MM-DD |
-->

## State Files Index

| Path | Purpose | Created at |
|---|---|---|
| `.planning/PROJECT.md` | Project vision, stack, constraints | STEP 0 |
| `.planning/REQUIREMENTS.md` | REQ-IDs với phase mapping | STEP 0 / Updated STEP 2 |
| `.planning/ROADMAP.md` | Milestones & sub-milestones | STEP 0 / Updated STEP 11 |
| `.planning/STATE.md` | Current position, next action, key decisions | STEP 0 / Updated mỗi step |
| `.planning/config.json` | Workflow config (mode, granularity, model defaults) | STEP 0 |
| `.planning/research/` | Research output dir (Mode B) | STEP 4 |
| `.planning/{phase}-RESEARCH.md` | Research output per phase (Mode A) | STEP 4 |
| `.planning/{phase}-{N}-PLAN.md` | Implementation plans (XML, wave-grouped) | STEP 6 |
| `.planning/{phase}-UAT.md` | User acceptance test results | STEP 8 |
| `.planning/{phase}-VERIFICATION.md` | Goal-backward verification | STEP 8 |
| `.planning/{phase}-SUMMARY.md` | Phase summary after ship | STEP 11 |
| `docs/specs/YYYY-MM-DD-{topic}-design.md` | Design specs (committed, human-readable) | STEP 5 |

## Tool Integrations

<!--
  Tool-managed zones go here. Pattern: tools update content between markers,
  do not edit manually. Example:

  <!-- gitnexus:start -->
  ... auto-generated by GitNexus ...
  <!-- gitnexus:end -->

  Add markers below as tools are wired in.
-->

## Last updated

{INIT_DATE}
