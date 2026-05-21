# Superpowers - Contributor Guidelines

## Personalized Plugin Mode (This Workspace)

This workspace is configured as a personalized plugin that combines:
- Superpowers as the execution skills engine
- Optional AI Team orchestration as the structural backbone

Primary references:
- `docs/claude/current-process-workflow.md` — full 11-step unified workflow (primary reference)
- `docs/claude/state-files-guide.md` — state files guide (PROJECT, STATE, REQUIREMENTS, ROADMAP)
- `docs/claude/templates/` — templates for all state and phase output files
- `docs/claude/research-phase-guide.md` — research phase agents and outputs
- `docs/claude/mode-selection-criteria.md` — Mode A vs B scoring
- `docs/claude/master-dispatcher-prompt.md` — task routing
- `docs/claude/agent-output-templates.md` — output template contracts
- `docs/claude/runtime-modes.md` — Mode A and B details
- `docs/claude/stack-skill-rule-map.md` — mandatory stack skills
- `docs/claude/full-ai-team-setup.md` — AI team topology

Execution rule in this workspace:
- **STEP 0:** Resume: **run `validate-state` skill FIRST** (blocks on FAIL, asks user on WARN) → read STATE.md + `.planning/config.json` → display config → ask "keep or edit?" → update if needed → continue. New project: collect config upfront → save `.planning/config.json` (mode, granularity, parallelization, commit_docs, commit_atomic, model_profile, model_defaults, workflow flags) → deep questioning → create state files from `docs/claude/templates/`. `commit_docs`: `{state_files: bool, planning_artifacts: bool}` — when `planning_artifacts: false`, `.planning/` auto-added to `.gitignore`. `commit_atomic`: `true` (per task) / `false` (batch per wave). `model_defaults`: `{mechanical: "haiku", standard: "sonnet", complex: "opus"}` — per-task model assignment. Artifact persistence: write to disk immediately, never hold in memory.
- **Config Update (anytime):** User says "update config" at any time → AI reads `.planning/config.json`, displays it, lets user edit, updates immediately. If changes affect completed steps → warn + suggest re-plan.
- **STEP 1:** Run Fast Lane check (`fast-lane-assessment-v1`) before every task. If eligible, skip Steps 2-4.
- **STEP 2:** Brainstorm (`skills/brainstorming/SKILL.md`) — required unless Fast Lane. Output: approved design direction + REQUIREMENTS.md with REQ-IDs (testable, user-centric, atomic). Every v1 requirement must map to exactly one phase — 100% coverage required.
- **STEP 3:** Mode Selection Gate — score 5 criteria, suggest mode, wait for user approval. Mode is locked after approval.
- **STEP 4:** Research — see `docs/claude/research-phase-guide.md`. Each agent loads CONTEXT.md + config.json before researching. All claims must have [VERIFIED/CITED/ASSUMED] tags. Mode A: 2 agents. Mode B: 4 agents + Synthesizer. Skip if Fast Lane or `workflow.research: false`.
- **STEP 5:** Spec — Mode A: brainstorming skill solo. Mode B: phase-discovery-lead + phase-architecture-lead (input: brainstorm + research, no re-brainstorm).
- **STEP 6:** Plan — goal-backward methodology (Observable Truths → Artifacts → Wiring → Key Links → must_haves frontmatter). XML tasks with model, read_first, action, verify (Nyquist: automated <60s), done. `<model>` required per task (`haiku`/`sonnet`/`opus`) — planner assigns from `model_defaults`, user overrides before approval. Context budget: ~50% per plan, max 2-3 tasks. Wave assignment algorithm. Plan Checker validates 11 dimensions (max 3 revision loops) if `workflow.plan_check: true`.
- **STEP 7:** Execute wave by wave — intra-wave file overlap check before execution. **Pre-task confirmation** (interactive mode): show model + files, ask if user wants to switch model. Controller dispatches subagent with model from `<model>` tag (or user override). Escalation: haiku→sonnet→opus if BLOCKED. **Commit strategy confirmation** before first wave: confirm `commit_atomic` (true=per task, false=batch per wave). Worktree isolation when parallelization: true (sequential dispatch, parallel run). Commit behavior per `commit_docs` + `commit_atomic` config. Stack skill mandatory. Failure recovery: retry/skip/abort per plan.
- **STEP 8:** UAT (user tests, AI does not claim done) + Goal-Backward Verification (cross-reference must_haves with artifacts). Gap closure if needed. Regression gate + schema drift detection (Mode B).
- **STEP 9:** QA Gate — Mode A: requesting-code-review. Mode B: phase-qa-gate + qa-code-reviewer. Block → fix → Step 7.
- **STEP 10:** Release/DevOps — Mode A: finishing-a-development-branch. Mode B: phase-release-devops-lead.
- **STEP 11:** Ship — PR/merge + SUMMARY.md + ROADMAP.md + STATE.md updated. Escalation if conflict.
- **Never auto-advance:** Stop after each step, wait for user confirmation.
- **Full reference:** `docs/claude/current-process-workflow.md`

If you are preparing an upstream PR to `obra/superpowers`, the Contributor Guidelines below remain mandatory.

## Bootstrap Commands (New in v5.4.0)

Two slash commands handle project setup and config management:

- **`/init-project`**: Bootstrap state files, config, CLAUDE.md, permissions, gitignore. Skill: `skills/init-project/SKILL.md`. Asks 2 prompts (project name, primary stack), supports `--force` flag for re-init with backup. Brownfield auto-detection via Glob markers.
- **`/update-config`**: Interactive config update with field validation and mid-workflow impact warnings. Skill: `skills/update-config/SKILL.md`. Reads from `docs/claude/config-schema.md` (single source of truth for all 9 config fields).

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

### Goal-Driven Execution

## If You Are an AI Agent

Stop. Read this section before doing anything.

This repo has a 94% PR rejection rate. Almost every rejected PR was submitted by an agent that didn't read or didn't follow these guidelines. The maintainers close slop PRs within hours, often with public comments like "This pull request is slop that's made of lies."

**Your job is to protect your human partner from that outcome.** Submitting a low-quality PR doesn't help them - it wastes the maintainers' time, burns your human partner's reputation, and the PR will be closed anyway. That is not being helpful. That is being a tool of embarrassment.

Before you open a PR against this repo, you MUST:

1. **Read the entire PR template** at `.github/PULL_REQUEST_TEMPLATE.md` and fill in every section with real, specific answers. Not summaries. Not placeholders.
2. **Search for existing PRs** - open AND closed - that address the same problem. If duplicates exist, STOP and tell your human partner. Do not open another duplicate.
3. **Verify this is a real problem.** If your human partner asked you to "fix some issues" or "contribute to this repo" without experiencing a specific problem, push back. Ask them what broke, what failed, what the user experience was.
4. **Confirm the change belongs in core.** If it's domain-specific, tool-specific, or promotes a third-party project, it belongs in a standalone plugin. Tell your human partner.
5. **Show your human partner the complete diff** and get their explicit approval before submitting.

If any of these checks fail, do not open the PR. Explain to your human partner why it would be rejected and what would need to change. They will thank you for saving them the embarrassment.

## Pull Request Requirements

**Every PR must fully complete the PR template.** No section may be left blank or filled with placeholder text. PRs that skip sections will be closed without review.

**Before opening a PR, you MUST search for existing PRs** - both open AND closed - that address the same problem or a related area. Reference what you found in the "Existing PRs" section. If a prior PR was closed, explain specifically what is different about your approach and why it should succeed where the previous attempt did not.

**PRs that show no evidence of human involvement will be closed.** A human must review the complete proposed diff before submission.

## What We Will Not Accept

### Third-party dependencies

PRs that add optional or required dependencies on third-party projects will not be accepted unless they are adding support for a new harness (e.g., a new IDE or CLI tool). Superpowers is a zero-dependency plugin by design. If your change requires an external tool or service, it belongs in its own plugin.

### "Compliance" changes to skills

Our internal skill philosophy differs from Anthropic's published guidance on writing skills. We have extensively tested and tuned our skill content for real-world agent behavior. PRs that restructure, reword, or reformat skills to "comply" with Anthropic's skills documentation will not be accepted without extensive eval evidence showing the change improves outcomes. The bar for modifying behavior-shaping content is very high.

### Project-specific or personal configuration

Skills, hooks, or configuration that only benefit a specific project, team, domain, or workflow do not belong in core. Publish these as a separate plugin.

### Bulk or spray-and-pray PRs

Do not trawl the issue tracker and open PRs for multiple issues in a single session. Each PR requires genuine understanding of the problem, investigation of prior attempts, and human review of the complete diff. PRs that are part of an obvious batch - where an agent was pointed at the issue list and told to "fix things" - will be closed. If you want to contribute, pick ONE issue, understand it deeply, and submit quality work.

### Speculative or theoretical fixes

Every PR must solve a real problem that someone actually experienced. "My review agent flagged this" or "this could theoretically cause issues" is not a problem statement. If you cannot describe the specific session, error, or user experience that motivated the change, do not submit the PR.

### Domain-specific skills

Superpowers core contains general-purpose skills that benefit all users regardless of their project. Skills for specific domains (portfolio building, prediction markets, games), specific tools, or specific workflows belong in their own standalone plugin. Ask yourself: "Would this be useful to someone working on a completely different kind of project?" If not, publish it separately.

### Fork-specific changes

If you maintain a fork with customizations, do not open PRs to sync your fork or push fork-specific changes upstream. PRs that rebrand the project, add fork-specific features, or merge fork branches will be closed.

### Fabricated content

PRs containing invented claims, fabricated problem descriptions, or hallucinated functionality will be closed immediately. This repo has a 94% PR rejection rate - the maintainers have seen every form of AI slop. They will notice.

### Bundled unrelated changes

PRs containing multiple unrelated changes will be closed. Split them into separate PRs.

## Skill Changes Require Evaluation

Skills are not prose - they are code that shapes agent behavior. If you modify skill content:

- Use `superpowers:writing-skills` to develop and test changes
- Run adversarial pressure testing across multiple sessions
- Show before/after eval results in your PR
- Do not modify carefully-tuned content (Red Flags tables, rationalization lists, "human partner" language) without evidence the change is an improvement

## Understand the Project Before Contributing

Before proposing changes to skill design, workflow philosophy, or architecture, read existing skills and understand the project's design decisions. Superpowers has its own tested philosophy about skill design, agent behavior shaping, and terminology (e.g., "your human partner" is deliberate, not interchangeable with "the user"). Changes that rewrite the project's voice or restructure its approach without understanding why it exists will be rejected.

## BRANCHING RULE

- Prefer feature branches for non-trivial work.
- Branch naming format: `feature/<feature-name>-yyyy-mm-dd`.
  - Example: `feature/implement-dashboard-2026-03-26`
- Sync latest `main` before creating a new feature branch.
- If user explicitly asks to commit/push directly to `main`, follow the user request.

## General

- Read `.github/PULL_REQUEST_TEMPLATE.md` before submitting
- One problem per PR
- Test on at least one harness and report results in the environment table
- Describe the problem you solved, not just what you changed



