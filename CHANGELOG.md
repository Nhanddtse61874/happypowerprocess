# Changelog

## [6.0.0] - 2026-07-11

### Added

- **Process 2.0 — three opt-in, default-off capabilities**, each gated by a `workflow.*` flag and built entirely from existing Claude Code primitives (markdown memory files, the `SessionStart` hook, the STEP 7 controller loop, the Bash tool). No external runtime, no new dependency. The 11-step workflow is unchanged until a flag is enabled.
  - **`workflow.memory_recall`** (MEM-01/02/03/04) — a process-memory store at `.claude/memory/` (committed to git) for "how we worked" lessons. STEP 0 Resume recalls the top-relevant entries on demand (grep/glob search-then-load, top-K ranked — not a full-index load) before the config prompt; STEP 11 writes new lessons back, English-normalized. Lessons carry a `scope: global | project` tag so per-project entries never bloat the global index. Boundary rule: process-memory captures "how we worked"; the knowledge library (`.claude/skills/<project>-*`) stays authoritative for "what the code IS" — process-memory never writes into it.
  - **`workflow.telemetry`** (OBS-01/02) — the STEP 7 controller appends per-task metrics (`model`, `wall_clock`, `escalation`, `req_ids`, and `tokens` when the harness exposes it) to `.planning/{phase}-telemetry.jsonl` (append-only, crash-tolerant); STEP 11 `finishing-a-development-branch` aggregates them into a telemetry table in the phase SUMMARY.
  - **`workflow.sandbox_verify`** (VER-01/02) — STEP 8 runs a bounded, evidence-only execute→observe→fix loop: hard cap of 3 fix iterations, each fix scope-fenced to the task's REQ-ID and planned files, no permission escalation. Run evidence lands in the `phase-VERIFICATION.md` Evidence column, separate from UAT; STEP 8a UAT and the human PASS/PARTIAL/FAIL decision stay unskippable — verify never auto-passes the step.
- **`docs/claude/memory-store-guide.md`** (NEW) — schema/contract for the `.claude/memory/` store: location, file format (MEMORY.md index + one `.md` per lesson), and the boundary rule vs the knowledge library.
- **`skills/memory-maintenance/SKILL.md`** (NEW) — scans the process-memory store and **flags** duplicate / stale / superseded / contradictory lessons for human-approved removal (never auto-deletes). Runs via `/memory-clean` or as an inline flag-and-report pass at STEP 11.
- **`commands/memory-clean.md`** (NEW) — slash-command registration for the maintenance skill.
- **Three config flags** — `workflow.memory_recall`, `workflow.telemetry`, `workflow.sandbox_verify` added to `docs/claude/config-schema.md` (schema rows + display-grouping indices 11/12/13) and `docs/claude/templates/config.json`, all default **`false`** (CFG-01).

### Pillars preserved

- Never auto-advance · REQ-ID 100% traceability · AI does not self-declare done. Hooks only inject reminders; recall/telemetry are passive reads/writes; sandbox verify is bounded and evidence-only; memory maintenance flags but never deletes.

### Backward compatibility

- 100% — every capability defaults off, so projects on prior versions behave identically until a flag is flipped. Memory recall, telemetry, and the maintenance pass are all no-ops when their flag is off or their store is absent.

## [5.8.0] - 2026-05-21

### Added

- **`skills/validate-state/SKILL.md`** (NEW) — safety-net skill that runs at STEP 0 of every session resume (and on-demand). Performs 4 categories of checks against state files:
  - **Category 1 — Existence**: PROJECT.md, STATE.md, REQUIREMENTS.md, ROADMAP.md, `.planning/config.json` all present.
  - **Category 2 — Schema**: STATE.md sections + status enum (`in_progress | blocked | waiting_for_user | complete`) + date format (`YYYY-MM-DD`); REQUIREMENTS.md REQ-ID pattern (`^[A-Z]{2,8}-\d{2,4}$`) + Phase assignment + no duplicates; ROADMAP.md + PROJECT.md required sections.
  - **Category 3 — Freshness**: STATE.md `last_updated` vs latest `.planning/` artifact mtime + latest non-state-file commit; WARN if ≥1 day stale.
  - **Category 4 — Drift**: cross-reference STATE.md current_step against artifact existence (PLAN.md, SUMMARY.md); REQ-ID phase coverage; ROADMAP/STATE milestone alignment; Mode consistency.
- Verdict semantics: **CLEAN** (continue), **WARN** (display report, user picks `accept_drift / fix_now / cancel`), **FAIL** (block until fixed).
- Cross-platform: skill describes logic only, AI translates to available tools per harness (no hardcoded shell).

### Changed (Layer 1 — preventive updates)

- **`skills/brainstorming/SKILL.md`** — added "Update STATE.md (mandatory at end)" section. Updates Phase, Status, Last updated, Next Action, Approved Mode, Key Decisions. Also refreshes REQUIREMENTS.md if drift detected at STEP 0.
- **`skills/writing-plans/SKILL.md`** — added "Update STATE.md (mandatory at end)" section. Updates STATE.md after plan written + ROADMAP.md milestone entry.
- **`skills/executing-plans/SKILL.md`** — added "Update STATE.md (mandatory after each wave/batch)" section. STATE.md `Last updated` is the trigger validate-state freshness check looks for.
- **`skills/subagent-driven-development/SKILL.md`** — added "Update STATE.md (mandatory after each task)" section. Continuous execution exception explicitly noted (file write does not pause execution).
- **`skills/finishing-a-development-branch/SKILL.md`** — added "Update STATE.md + ROADMAP.md + SUMMARY (mandatory at end)" section. Closes the work cycle by finalizing all 3 state files.

### Changed (Layer 3 — workflow integration)

- **`CLAUDE.md`** STEP 0 — now mandates `validate-state` skill BEFORE reading STATE.md on resume.
- **`docs/claude/current-process-workflow.md`** STEP 0 "Resuming session" — explicit `validate-state` invocation as first action.
- **`docs/claude/state-files-guide.md`** — added "STATE.md update chain (Layer 1)" table mapping each skill to STATE.md fields it owns; added "Validation (Layer 2)" + "Cross-platform safety net (Layer 3)" sections.
- **`hooks/session-start`** — detects `STATE.md` in cwd; injects `<project-state-reminder>` block instructing AI to run validate-state before any work. Works on Claude Code + Cursor + Copilot CLI. Codex falls back to CLAUDE.md directive (no hook support).

### Verification

- E2E tested in `C:\temp\hppp-validate-test-2026-05-21\` (cleaned up after) across 4 scenarios:
  - **Scenario A (clean baseline)**: all 4 categories PASS → verdict CLEAN
  - **Scenario B (schema error)**: Status `inprogress` (typo) + date `2026/5/21` (wrong format) → verdict FAIL, 2 specific fixes suggested
  - **Scenario C (freshness drift)**: STATE.md `last_updated` 2026-05-10 vs code commit 2026-05-21 (11-day gap) → verdict WARN with drift detail
  - **Scenario D (cross-ref drift)**: STATE.md says Step 5 but `M1-SUMMARY.md` exists; ROADMAP says in_progress but SUMMARY claims complete → verdict WARN with 2 contradictions

### Why

Resolves Weakness #4 from the v5.7.0 retrospective: "State files cần maintain bằng tay — không có skill auto-update STATE.md/ROADMAP.md sau mỗi wave". 4-layer defense in depth:
- **Layer 1 (prevention)**: every major skill now updates STATE.md before exit
- **Layer 2 (detection)**: validate-state runs at STEP 0 to catch any update that was missed
- **Layer 3 (safety net)**: SessionStart hook + CLAUDE.md directive ensure Layer 2 fires on every session resume
- **Layer 4 (schema enforcement)**: DEFERRED to v5.9+ (YAML frontmatter in STATE.md)

### Plugin manifests

- All 6 declared files bumped 5.7.0 → 5.8.0.
- README.md + `.opencode/INSTALL.md` updated.

### Backward compatibility

- No breaking change. Existing projects with stale STATE.md will get WARN on next session resume → user chooses `accept_drift / fix_now`. No state file is modified without user consent.
- New skill is opt-in via STEP 0; projects that don't run validate-state continue to work as before.
- Skills with new "Update STATE.md" sub-step still work if STATE.md is absent (no-op).
- Hook gracefully no-ops if STATE.md not in cwd.

## [5.7.0] - 2026-05-21

### Added

- **`skills/fast-lane-assessment-v1/SKILL.md`** — new runnable skill that fills the gate referenced by STEP 1 of CLAUDE.md. Previously, the workflow named `fast-lane-assessment-v1` as a check to run before brainstorming but no SKILL.md existed, so the gate was ad-hoc per session. The new skill references existing sources of truth without duplication: criteria live in `docs/claude/current-process-workflow.md` STEP 1, output schema lives in `docs/claude/agent-output-templates.md` T6. The skill orchestrates: score five criteria + hard exclusions → emit JSON → user confirmation.

### Changed

- **`skills/using-git-worktrees/SKILL.md`** — synced upstream improvements while preserving the fork's `## Integration` block. New: Step 0 isolation detection (`GIT_DIR` vs `GIT_COMMON` with submodule guard via `--show-superproject-working-tree`), Step 1a native-tool preference (e.g., `EnterWorktree` over `git worktree add`), Step 1b git fallback, sandbox-permission fallback. Expanded Quick Reference (13 rows), new Common Mistakes (Fighting the harness, Skipping detection), new Red Flags (creating a worktree when already isolated, using `git worktree add` when a native tool exists).
- **`skills/finishing-a-development-branch/SKILL.md`** — synced upstream improvements while preserving the fork's `## Integration` block. New: Step 2 environment detection (normal repo / named-branch worktree / detached HEAD), detached-HEAD 3-option menu (no merge), Step 5 CWD safety (`cd MAIN_ROOT` before merge/discard), Step 6 provenance-based cleanup (only remove worktrees under `.worktrees/`, `worktrees/`, `~/.config/superpowers/worktrees/` — harness-owned workspaces left alone). New Common Mistakes (deleting branch before worktree removal, running `git worktree remove` from inside the worktree, cleaning up harness-owned worktrees) and Red Flags (#1: remove a worktree before confirming merge success).
- **`skills/subagent-driven-development/SKILL.md`** — ported upstream's `## Continuous execution` paragraph: do not pause between tasks for "should I continue?" prompts. The fork's Pre-Task Confirmation, `<model>` XML tag dispatch rule, `.planning/config.json` `model_defaults` fallback, and BLOCKED escalation tiers are preserved; an explicit exception clause notes that Pre-Task Confirmation still runs unless `mode: yolo` is set in `.planning/config.json`.

### Verified upstream-identical (no action)

- `using-superpowers`, `writing-skills`, `dispatching-parallel-agents`, `systematic-debugging`, `test-driven-development`, `verification-before-completion`, `receiving-code-review` — diff-confirmed byte-identical to `obra/superpowers` main.

### Verified fork-ahead (intentionally retained, no upstream sync)

- `brainstorming` — keeps Mode Selection Gate section + HARD-GATE block + checklist item 9 (Mode B integration).
- `writing-plans` — keeps Mode B path callout (phase-discovery-lead / phase-architecture-lead / phase-implementation-lead routing).
- `executing-plans` — keeps stricter "REQUIRED: Set up isolated workspace before starting" wording (matches STEP 7 worktree enforcement).
- `requesting-code-review` — keeps `superpowers:code-reviewer` agent type reference (fork agent at `agents/code-reviewer.md`); upstream switched to `general-purpose` but our agent is real.

### Verification

- E2E tested on a mock `todo-cli` project in `C:\temp\hppp-mock-test-2026-05-21\` (cleaned up after): Fast Lane eligible scenario → ELIGIBLE verdict, non-eligible scenario → NOT ELIGIBLE verdict, Step 0 detection in normal repo + from inside linked worktree, real CRUD implementation (9/9 tests pass), Step 2 environment detection identifies named-branch worktree, Option 4 provenance cleanup removes worktree and branch without touching anything outside known paths.

### Plugin manifests

- All 6 declared files bumped 5.6.1 → 5.7.0: `package.json`, `.claude-plugin/plugin.json`, `.cursor-plugin/plugin.json`, `.claude-plugin/marketplace.json`, root `marketplace.json`, `gemini-extension.json`.
- `README.md` version badge + structure note, and `.opencode/INSTALL.md` pinned-version example, bumped to 5.7.0.

### Backward compatibility

- No config schema changes. Existing `.planning/config.json` files continue to work as-is.
- No breaking changes to Mode A or Mode B. The 11-step workflow contract is preserved; STEP 1 now resolves to a real skill instead of a missing reference.
- Worktree skill changes are additive — projects already using `.worktrees/` continue to work; the new Step 0 detection short-circuits creation when already isolated rather than erroring.

## [5.6.1] - 2026-05-07

### Policy

- **English-only plugin source policy**: all plugin source content (skills, slash commands, templates, schema docs, design specs, workflow docs) must be written in English. The user-facing AI conversation language is independent and follows the user's input language. Any non-English string hard-coded into a plugin file is now treated as a defect; localization belongs in `.planning/config.json` per project, not in plugin source.

### Fixed

- **Skill prompts** — translated 4 mixed-language prompts to English in `skills/init-project/SKILL.md` (idempotency menu choice [1] hint, git-init missing-repo message), `skills/update-config/SKILL.md` (field-pick prompt), and `skills/add-tech-stack/SKILL.md` (AGENT identity prompt).

### Changed

- **State templates** — full English rewrite of `docs/claude/templates/PROJECT.md`, `REQUIREMENTS.md`, `STATE.md`, `phase-SUMMARY.md`. Prior templates contained Vietnamese placeholder hints that bled into every initialized project.
- **`docs/claude/templates/CLAUDE.md`** — Workflow Mandate, Plugin References, Stack-Specific Rules, Project-Specific Deviations, and State Files Index sections translated to English.
- **`docs/claude/workflow-diagram.md`** — Mermaid node labels (S0 Bootstrap, S8 UAT) and the v1-vs-v2 comparison table translated to English. Preview instruction line translated.
- **`docs/specs/2026-05-06-init-project-skill-design.md`** — full English rewrite (Problem Statement, Goals, Architecture, Components, Data Flow, Error Handling, Testing Approach, Decisions Log).
- **`docs/specs/2026-05-07-stack-customization-v5.5.0-design.md`** — translated 4 Vietnamese strings (CLAUDE.md marker block sample, Q-identity / Q-goal prompt examples).

### Plugin manifests

- All 5 manifests bumped 5.6.0 → 5.6.1: `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.cursor-plugin/plugin.json`, root `marketplace.json`, `package.json`.
- `gemini-extension.json` and pinned-version examples in `.opencode/INSTALL.md` and `README.md` bumped to 5.6.1.

### Backward compatibility

- Existing initialized projects (`.planning/*` already created) are unaffected — old state files with Vietnamese content remain as-is. Only newly initialized projects pick up the English templates.
- No config schema changes. No behavior changes in skills beyond prompt wording.

## [5.6.0] - 2026-05-07

### Policy Change

- **Mode B (Team Spine) is now officially Claude Code-only**. On Cursor / Codex / OpenCode, Mode Selection Gate auto-forces Mode A regardless of task complexity score. Rationale: Mode B requires Claude-specific Task tool features (subagent dispatch with `subagent_type` + `model` + `isolation` parameters; Claude model names like `haiku` / `sonnet` / `opus`; worktree-isolated parallel execution). Previously Mode B would partial-fail on non-Claude harnesses.

### Added

- **`active_harness` config field** (enum: `auto` / `claude-code` / `cursor` / `codex` / `opencode`, default `auto`). Detected via env vars at workflow time; fallback `claude-code` preserves existing behavior.
- **`docs/claude/harness-compatibility.md`** — comprehensive matrix (4 harnesses × 2 modes), rationale, workaround guidance for non-Claude users, detection mechanism, user-override semantics.

### Changed

- **`docs/claude/mode-selection-criteria.md`** — adds Reverse Hard Exclusion section: non-Claude harness → force Mode A. User can still override per existing user-decision-wins rule, with hard warning displayed.
- **`docs/claude/templates/CLAUDE.md`** — adds 1-line Mode B Claude-only note in Workflow Mandate section.
- **`docs/claude/templates/config.json`** — adds `active_harness: "auto"` as first field.
- **`docs/claude/config-schema.md`** — adds `active_harness` row + index 0 in Display Grouping.

### Backward compatibility

- v5.5.x projects continue to work unchanged. `active_harness` auto-resolves via env detection at next Mode Gate; explicit field added at next `/update-config` invocation.
- Existing plans, snapshots, registry — no changes required.
- Power users on non-Claude harnesses can still override Mode Gate decision (consistent with project's user-decision-wins principle).

### Strategy

Chose **Strategy α (Runtime enforcement)** over **Strategy β (Per-marketplace bundles)**: 10x cheaper, single source of truth, eliminates drift class of bugs (v5.4.0 marketplace.json drift case). β rejected as over-engineering for project size.

## [5.5.1] - 2026-05-07

### Fixed

- **Cross-platform plugin path detection** — `/init-project`, `/update-config`, `/add-tech-stack`, `/sync-stack-skill` now work on Codex / Cursor / OpenCode (not just Claude Code). Skill bodies try 4 candidate dirs in priority order: `~/.claude/plugins/marketplaces/`, `~/.codex/`, `~/.cursor/extensions/`, `~/.opencode/plugins/`, with multi-harness Glob fallback.
- **Codex INSTALL.md** — install steps now also symlink `agents/` (for Mode B users) plus note that Codex doesn't register slash commands — invoke skills by name instead.
- **Cursor plugin.json keyword drift** — synced with Claude plugin.json (added 4 missing: `research-phase`, `wave-execution`, `goal-backward-planning`, `runtime-modes`).

### Plugin manifests

- All 6 manifests bumped 5.5.0 → 5.5.1.

## [5.5.0] - 2026-05-07

### Added

- **`/add-tech-stack`** ([skills/add-tech-stack/SKILL.md](skills/add-tech-stack/SKILL.md)) — customize an existing plugin stack skill for the current project, or add a brand-new stack (Python, Go, Rust, etc.). Two flows: Scenario A (existing — keep/override/append/skip per section) and Scenario B (new — input/suggest/skip + variable section loop). Force both SKILL.md + AGENT.md when creating new; ask user file choice when editing existing.
- **`/sync-stack-skill`** ([skills/sync-stack-skill/SKILL.md](skills/sync-stack-skill/SKILL.md)) — three-way merge customized snapshot with latest plugin default. Per-section conflict detection with 4 patterns (both unchanged / plugin-only / user-only / both updated); manual-merge sub-flow with 3-pane view. Supports `--all` for batch sync.
- **Architecture detection (T1 sub-flow)** in `/init-project` — opt-in heuristic detection of project's actual architecture (Clean / Vertical Slices / N-Tier / Feature-folders / Atomic) for brownfield projects. E3 conservative — every step requires explicit user consent.
- **Stack registry** at `.claude/stack-skills/registry.json` — single source of truth for registered stacks per project. Default 5 plugin stacks + user-added stacks.
- **Templates**: `docs/claude/templates/registry.json`, `stack-skill-template.md`, `stack-agent-template.md` — used by `/init-project` and `/add-tech-stack` Scenario B.

### Changed

- **`docs/claude/templates/CLAUDE.md`** — Stack-Specific Rules table now wrapped in `<!-- stack-table:start --> ... <!-- stack-table:end -->` markers; auto-regenerated by `/add-tech-stack` and `/sync-stack-skill` from registry.
- **`skills/init-project/SKILL.md`** — adds T1 architecture detection sub-flow (section d2) and creates `.claude/stack-skills/registry.json` from template at init time.
- **`docs/claude/stack-skill-rule-map.md`** — runtime lookup now consults `.claude/stack-skills/registry.json`; T2 unknown stack mid-task halt behavior documented.

### Plugin manifests

- `.claude-plugin/plugin.json` 5.4.0 → 5.5.0; keywords added: `add-tech-stack`, `sync-stack-skill`, `stack-customization`.
- `.cursor-plugin/plugin.json` 5.4.0 → 5.5.0; same keyword additions.
- `.claude-plugin/marketplace.json` 5.4.0 → 5.5.0.

### Backward compatibility

- v5.4.0 projects continue to work unchanged. Stack registry is auto-created at first `/add-tech-stack` or `/sync-stack-skill` invocation if missing. CLAUDE.md without markers prompts user to refresh template.

## [5.4.0] - 2026-05-06

### Added

- **`/init-project` skill** — bootstrap project state files, config, CLAUDE.md, permissions, and gitignore. Asks 2 minimal prompts (project name + primary stack), supports `--force` flag for re-init with backup. Brownfield auto-detection via Glob markers (`*.csproj`, `package.json`, `Cargo.toml`, etc.). Idempotency: 4-choice menu when STATE.md exists (abort / update-config / re-init missing / force re-init). Cross-platform plugin path detection (Windows + macOS/Linux) with Glob fallback for non-marketplace installs.
- **`/update-config` skill** — interactive config update with field validation against `docs/claude/config-schema.md` (single source of truth). Per-field validation, mid-workflow impact warnings, summary diff before apply. Auto-updates `.gitignore` when `commit_docs.planning_artifacts` toggles. Trigger phrases: "update config", "change config", "change model", "change commit strategy".
- **`docs/claude/config-schema.md`** — single source of truth for 9 config fields (12 rendered rows: nested `commit_docs.*`, `model_defaults.*`, `workflow.*`). Documents valid values, defaults (must match `templates/config.json`), mid-workflow impact, display grouping convention, and Read-permission trade-off.
- **`docs/claude/templates/CLAUDE.md`** — project-level CLAUDE.md template (consumed by `/init-project`). Workflow mandate without "Superpowers" naming, plugin path placeholders for cross-platform, embedded config schema in STEP 0, full STEP 7 operational details, Stack-Specific Rules table, Project-Specific Deviations table, State Files Index.
- **`docs/claude/templates/settings.json`** — Option C workflow-aware permission allowlist (read-only ops pre-allowed, writes/destructive require user approval).
- **`docs/claude/templates/gitignore`** — base entries: `.worktrees/`, `.claude/settings.local.json`. The `.planning/` line is added/removed at runtime by `/update-config` per `commit_docs.planning_artifacts` toggle.
- **`commands/init-project.md`** + **`commands/update-config.md`** — slash command registrations for cross-harness compatibility.

### Changed

- **Plugin manifests** — added 4 new keywords to both `.claude-plugin/plugin.json` and `.cursor-plugin/plugin.json`: `init-project`, `bootstrap`, `config-management`, `project-setup`.
- **Plugin CLAUDE.md** — new "Bootstrap Commands (New in v5.4.0)" section documents both new slash commands.

### Fixed

- **Marketplace version drift** — `.claude-plugin/marketplace.json` was at `5.2.0` while other manifests at `5.3.0`. Bumped all to `5.4.0` in this release, restoring sync.

## [5.3.0] - 2026-04-13

### Added

- **Version skill** — `/version` command reads `package.json` and displays current plugin version
- **README rewrite** — 17-section structure with TOC, badges, project structure, configuration guide, prerequisites, troubleshooting, contributing, attribution
- **Per-task model assignment** — `<model>` element in plan XML (`haiku`/`sonnet`/`opus`) with `model_defaults` in config
- **Granular commit control** — `commit_docs` object (`state_files` + `planning_artifacts`), `commit_atomic` (per task vs per wave)
- **Config management** — "update config" anytime, resume session config display with keep/edit prompt
- **Pre-task confirmation** — Step 7 shows model + files before each task

### Changed

- **Version source of truth** — `package.json` is now the canonical version reference
- **CHANGELOG backfill** — added v5.1.0 and v5.2.0 entries from RELEASE-NOTES

### Fixed

- Removed redundant config fields (`verifier`, `nyquist_validation`, `auto_advance`)

## [5.2.0] - 2026-04-08

### Added

- **11-step unified workflow** — GSD state persistence + research phase + wave-based execution + UAT gate combined with Superpowers Mode A/B + QA gate + DevOps phase
- **State files** — PROJECT.md, STATE.md, REQUIREMENTS.md, ROADMAP.md, `.planning/config.json` persist across sessions
- **Research phase** — Mode A (2 agents) / Mode B (4 agents + Synthesizer) with [VERIFIED/CITED/ASSUMED] claim provenance
- **Goal-backward planning** — Observable Truths → Required Artifacts → Required Wiring → must_haves frontmatter
- **Nyquist Rule** — automated verify steps must complete in <60s
- **Plan Checker** — 11-dimension validation before execution
- **Wave execution** — intra-wave file overlap check, fresh context per task, atomic commits, worktree isolation
- **10 human touchpoints** — AI never auto-advances without user confirmation
- **marketplace.json** — plugin marketplace install support

### Changed

- **Implementer skills rewritten** — React/TS (TanStack Query v6, Zustand, RHF+Zod), Angular/TS (Signals, RxJS takeUntilDestroyed, OnPush), IoT Edge (MQTT QoS, LWT, BLE GATT, TLS/mTLS), React Native/TS (typed navigation, MMKV, CodePush)
- **All 20 agents aligned** with unified workflow (REQ-ID format, goal-backward methodology, STATE.md updates)
- **Plugin renamed** to `happypowerprocess`
- **All platform configs updated** — Claude Code, Cursor, Codex, OpenCode, Gemini

## [5.1.0] - 2026-04-06

### Added

- **Initial personalized fork** from obra/superpowers
- **AI Team orchestration agents** — phase leads, orchestrator, dispatcher
- **GSD workflow documentation** stubs
- **CLAUDE.md** with workspace-specific execution rules

## [5.0.5] - 2026-03-17

### Fixed

- **Brainstorm server ESM fix**: Renamed `server.js` → `server.cjs` so the brainstorming server starts correctly on Node.js 22+ where the root `package.json` `"type": "module"` caused `require()` to fail. ([PR #784](https://github.com/obra/superpowers/pull/784) by @sarbojitrana, fixes [#774](https://github.com/obra/superpowers/issues/774), [#780](https://github.com/obra/superpowers/issues/780), [#783](https://github.com/obra/superpowers/issues/783))
- **Brainstorm owner-PID on Windows**: Skip `BRAINSTORM_OWNER_PID` lifecycle monitoring on Windows/MSYS2 where the PID namespace is invisible to Node.js. Prevents the server from self-terminating after 60 seconds. The 30-minute idle timeout remains as the safety net. ([#770](https://github.com/obra/superpowers/issues/770), docs from [PR #768](https://github.com/obra/superpowers/pull/768) by @lucasyhzhu-debug)
- **stop-server.sh reliability**: Verify the server process actually died before reporting success. Waits up to 2 seconds for graceful shutdown, escalates to `SIGKILL`, and reports failure if the process survives. ([#723](https://github.com/obra/superpowers/issues/723))

### Changed

- **Execution handoff**: Restore user choice between subagent-driven-development and executing-plans after plan writing. Subagent-driven is recommended but no longer mandatory. (Reverts `5e51c3e`)
