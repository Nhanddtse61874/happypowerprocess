# happypowerprocess — Workspace Instructions

> Operational rules for working **in this repository**. For upstream PR / contribution rules (the 94%-rejection guardrails, PR template requirements, what won't be accepted), see [CONTRIBUTING.md](CONTRIBUTING.md).

## Personalized Plugin Mode (This Workspace)

This workspace is configured as a personalized plugin that combines:
- Superpowers as the execution skills engine
- Optional AI Team orchestration as the structural backbone

Primary references:
- `docs/claude/current-process-workflow.md` — full 11-step unified workflow (primary reference)
- `docs/claude/config-schema.md` — all `.planning/config.json` fields + defaults (single source of truth)
- `docs/claude/state-files-guide.md` — state files guide (PROJECT, STATE, REQUIREMENTS, ROADMAP)
- `docs/claude/templates/` — templates for all state and phase output files
- `docs/claude/research-phase-guide.md` — research phase agents and outputs
- `docs/claude/modes.md` — runtime modes + Mode Selection Gate + harness compatibility
- `docs/claude/ai-team.md` — Mode B team topology + dispatch routing
- `docs/claude/agent-output-templates.md` — output template contracts
- `docs/claude/stack-skill-rule-map.md` — mandatory stack skills

Execution rule in this workspace:
- **STEP 0:** Resume: **run `validate-state` skill FIRST** (blocks on FAIL, asks user on WARN) → read STATE.md + `.planning/config.json` → display config → ask "keep or edit?" → update if needed → continue. New project: collect config upfront → save `.planning/config.json` (all fields + defaults: see `docs/claude/config-schema.md`) → deep questioning → create state files from `docs/claude/templates/`. Artifact persistence: write to disk immediately, never hold in memory.
- **Config Update (anytime):** User says "update config" at any time → AI reads `.planning/config.json`, displays it, lets user edit, updates immediately. If changes affect completed steps → warn + suggest re-plan.
- **STEP 0.5 (optional) — Knowledge Bootstrap:** for a complex/brownfield project, optionally build a per-project knowledge library (`.claude/skills/<project>-*`) via the `mining-project-knowledge` skill (`/mine-knowledge`) — once per project. When a library exists, STEP 1/4/7/9/11 consume and maintain it via the conditional hooks below (each is a no-op without a library).
- **STEP 1:** Run Fast Lane check (`fast-lane-assessment-v1`) before every task. If eligible, skip Steps 2-4. Hook: if `<project>-change-control` flags the touched area dangerous → not Fast-Lane-eligible.
- **STEP 2:** Brainstorm (`skills/brainstorming/SKILL.md`) — required unless Fast Lane. Output: approved design direction + REQUIREMENTS.md with REQ-IDs (testable, user-centric, atomic). Every v1 requirement must map to exactly one phase — 100% coverage required.
- **STEP 3:** Mode Selection Gate — score 5 criteria, suggest mode, wait for user approval. Mode is locked after approval. See `docs/claude/modes.md`.
- **STEP 4:** Research — see `docs/claude/research-phase-guide.md`. Each agent loads CONTEXT.md + config.json before researching. All claims must have [VERIFIED/CITED/ASSUMED] tags. Mode A: 2 agents. Mode B: 4 agents + Synthesizer. Skip if Fast Lane or `workflow.research: false`. Hook: if `<project>-*` knowledge skills exist, load them FIRST and research only the gaps.
- **STEP 5:** Spec — Mode A: brainstorming skill solo. Mode B: phase-discovery-lead + phase-architecture-lead (input: brainstorm + research, no re-brainstorm).
- **STEP 6:** Plan — goal-backward methodology (Observable Truths → Artifacts → Wiring → Key Links → must_haves frontmatter). XML tasks with model, read_first, action, verify (Nyquist: automated <60s), done. `<model>` required per task (`haiku`/`sonnet`/`opus`) — planner assigns from `model_defaults`, user overrides before approval. Context budget: ~50% per plan, max 2-3 tasks. Wave assignment algorithm. Plan Checker validates 11 dimensions (max 3 revision loops) if `workflow.plan_check: true`.
- **STEP 7:** Execute wave by wave — intra-wave file overlap check before execution. **Pre-task confirmation** (interactive mode): show model + files, ask if user wants to switch model. Controller dispatches subagent with model from `<model>` tag (or user override). Escalation: haiku→sonnet→opus if BLOCKED. **Commit strategy confirmation** before first wave: confirm `commit_atomic` (true=per task, false=batch per wave). Worktree isolation when parallelization: true (sequential dispatch, parallel run). Commit behavior per `commit_docs` + `commit_atomic` config. Stack skill mandatory. Failure recovery: retry/skip/abort per plan. Hook: on BLOCKED, consult `<project>-debugging-playbook` (if present) before escalating model.
- **STEP 8:** UAT (user tests, AI does not claim done) + Goal-Backward Verification (cross-reference must_haves with artifacts). Gap closure if needed. Regression gate + schema drift detection (Mode B).
- **STEP 9:** QA Gate — Mode A: requesting-code-review. Mode B: phase-qa-gate + qa-code-reviewer. Block → fix → Step 7. Hook: use `<project>-validation-and-qa` (if present) as the evidence bar.
- **STEP 10:** Release/DevOps — Mode A: finishing-a-development-branch. Mode B: phase-release-devops-lead.
- **STEP 11:** Ship — PR/merge + SUMMARY.md + ROADMAP.md + STATE.md updated. Escalation if conflict. Knowledge Sync hook (if a `<project>-*` library exists): write lessons back — new bug → `failure-archaeology` + `debugging-playbook`; new config → `config-and-flags`; new decision → `architecture-contract`.
- **Never auto-advance:** Stop after each step, wait for user confirmation.
- **Full reference:** `docs/claude/current-process-workflow.md`

## Bootstrap Commands (New in v5.4.0)

Two slash commands handle project setup and config management:

- **`/init-project`**: Bootstrap state files, config, CLAUDE.md, permissions, gitignore. Skill: `skills/init-project/SKILL.md`. Asks 2 prompts (project name, primary stack), supports `--force` flag for re-init with backup. Brownfield auto-detection via Glob markers.
- **`/update-config`**: Interactive config update with field validation and mid-workflow impact warnings. Skill: `skills/update-config/SKILL.md`. Reads from `docs/claude/config-schema.md` (single source of truth for all config fields).

Both skills follow the 11-step workflow: `/init-project` produces STEP 0 artifacts, `/update-config` is invoked by user trigger phrases at any time.

Schema reference: `docs/claude/config-schema.md`

## Stack Customization Commands (New in v5.5.0)

Two slash commands handle per-project stack skill customization and sync:

- **`/add-tech-stack`**: Customize an existing plugin stack skill for the current project, or add a brand-new stack (Python, Go, Rust, etc.). Skill: `skills/add-tech-stack/SKILL.md`. Two scenarios:
  - **Scenario A — existing stack**: registered already (default 5 or previously added). Walks each section with `[keep / override / append / skip]`. User picks file scope: SKILL only / AGENT only / both.
  - **Scenario B — new stack**: not registered. Forces both SKILL.md + AGENT.md. Walks template sections with `[input / suggest / skip]`. Best-practice MINIMUM filled on explicit skip.
  - Snapshot stored at `.claude/stack-skills/<stack>/SKILL.md` and `.claude/agents/implementer-<stack>.md`. Project snapshot wins over plugin default at runtime.

- **`/sync-stack-skill`**: Three-way merge a customized snapshot with the latest plugin default. Skill: `skills/sync-stack-skill/SKILL.md`. Per-section conflict detection (4 patterns); manual-merge 3-pane sub-flow. Supports `--all` for batch sync.

- **Architecture detection (T1)** integrates into `/init-project` brownfield flow — opt-in scan of project structure to populate the customized stack skill `## Architecture` section. E3 conservative — every step requires explicit user consent.

Both skills follow the 11-step workflow and use `.claude/stack-skills/registry.json` as the runtime source of truth. Schema reference: `docs/specs/2026-05-07-stack-customization-v5.5.0-design.md`.

## Code-Level Behavioral Guidelines

The 11-step workflow above defines WHAT/WHEN. This section defines HOW when actually touching code in STEP 7.

### Think Before Coding
- When code relies on an unverified assumption, tag `[ASSUMED]` in the commit message or PR description (extends the `[VERIFIED/CITED/ASSUMED]` mechanism from STEP 4 to code-level).
- If a REQ-ID has multiple interpretations, surface all of them — do not silently pick one.
- If a simpler approach than the approved plan exists, push back with justification before coding.

### Simplicity First
- Write the minimum code that satisfies the REQ-ID. Nothing speculative.
- No abstractions, flexibility, or configurability that were not requested.
- No error handling for impossible scenarios.
- Self-check: "Would a senior engineer call this overcomplicated?"
- If this conflicts with Plan Checker's 11 dimensions, prefer simplicity and flag it in the QA Gate (STEP 9).

### Surgical Changes
- Only modify what is strictly required for the current task/REQ-ID.
- Do not "improve" surrounding code, formatting, or comments.
- Do not refactor things that are not broken; match existing style.
- Unrelated dead code — mention it in the PR description, do not delete.
- Only remove imports/variables/functions that your own changes made orphan.
- Every changed line must trace back to a REQ-ID or a task in the plan.

## BRANCHING RULE

- Prefer feature branches for non-trivial work.
- Branch naming format: `feature/<feature-name>-yyyy-mm-dd`.
  - Example: `feature/implement-dashboard-2026-03-26`
- Sync latest `main` before creating a new feature branch.
- If user explicitly asks to commit/push directly to `main`, follow the user request.

## Contributing / Upstream PRs

If you are preparing an upstream PR to `obra/superpowers` (or a PR to this repo), the rules in [CONTRIBUTING.md](CONTRIBUTING.md) are mandatory: read the PR template, search existing PRs, one problem per PR, show your human partner the full diff first.
