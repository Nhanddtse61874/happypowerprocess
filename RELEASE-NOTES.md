# Release Notes — happypowerprocess

## v6.0.0 (2026-07-11)

### Process 2.0 — Memory Recall, Observability & Sandbox Verification

Process 2.0 absorbs the strengths of three agent stacks (Mem0, DeerFlow, MS Agent Framework) into the existing 11-step workflow as **native Claude Code primitives — no external runtime, no new dependency**. It ships **three opt-in capabilities, each gated by a `workflow.*` flag and defaulting to `false`**. The workflow is unchanged until you enable one.

- **`workflow.memory_recall`** — a process-memory store at `.claude/memory/` (committed to git) captures "how we worked" lessons. STEP 0 Resume recalls the top-relevant entries on demand (grep/glob, ranked, top-K — not a full-index load) before the config prompt; STEP 11 writes new lessons back (English-normalized). Distinct from the knowledge library, which stays authoritative for "what the code IS".
- **`workflow.telemetry`** — the STEP 7 controller appends per-task metrics (model, wall-clock, escalations, tokens when the harness exposes them) to `.planning/{phase}-telemetry.jsonl`; STEP 11 aggregates them into a telemetry table in the phase SUMMARY.
- **`workflow.sandbox_verify`** — STEP 8 runs a **bounded, evidence-only** execute→observe→fix loop (hard cap 3 iterations, scope-fenced to the task's REQ-ID and planned files, no permission escalation). It attaches run evidence to VERIFICATION **before** UAT and **never** auto-passes the step — UAT and the human PASS/PARTIAL/FAIL decision remain unskippable.

### Memory maintenance

- **`/memory-clean`** (skill `memory-maintenance`) — scans the process-memory store and **flags** duplicate / stale / superseded / contradictory lessons for **human-approved** removal (never auto-deletes). Runs on demand or as an inline flag-and-report pass at STEP 11.

### New files

- **`docs/claude/memory-store-guide.md`** — schema/contract for the `.claude/memory/` store: location, file format, and the boundary rule vs the knowledge library.
- **`skills/memory-maintenance/SKILL.md`** — the maintenance skill behind `/memory-clean` and the STEP-11 pass.
- **`commands/memory-clean.md`** — slash-command registration.
- **Three new config flags** — `workflow.memory_recall`, `workflow.telemetry`, `workflow.sandbox_verify` added to `config-schema.md` + `templates/config.json`, all default `false`.

### The 3 pillars are preserved

Never auto-advance · REQ-ID 100% traceability · AI does not self-declare done. Hooks only inject reminders; recall/telemetry are passive reads/writes; verify is bounded and evidence-only; maintenance flags but never deletes.

### Backward compatibility

- 100% — every capability defaults off. Projects on prior versions behave identically until a flag is flipped. Both memory recall and telemetry are no-ops when their store/flag is absent.

---

## v5.8.0 (2026-05-21)

### State Files Validation System (4-layer defense)

- **`validate-state` skill (NEW)** — runs at STEP 0 of every session resume (and on demand). Four check categories against state files: Existence, Schema (STATE.md sections + status enum + date format; REQUIREMENTS.md REQ-ID pattern + phase coverage + no dupes; ROADMAP/PROJECT sections), Freshness (STATE.md `last_updated` vs latest artifact/commit), and Drift (cross-reference current_step against artifacts, REQ-ID phase coverage, milestone alignment, mode consistency).
- **Verdicts** — **CLEAN** (continue), **WARN** (display report, user picks `accept_drift / fix_now / cancel`), **FAIL** (block until fixed). No state file is modified without user consent.

### Layer 1 — preventive STATE.md updates

- `brainstorming`, `writing-plans`, `executing-plans`, `subagent-driven-development`, and `finishing-a-development-branch` each gained a mandatory "Update STATE.md" sub-step, so state stays fresh after every brainstorm / plan / wave / task / branch close.

### Layer 3 — workflow integration

- **`hooks/session-start`** detects STATE.md in cwd and injects a `<project-state-reminder>` telling the AI to run `validate-state` first (Claude Code + Cursor + Copilot CLI; Codex falls back to the CLAUDE.md directive).
- CLAUDE.md, `current-process-workflow.md`, and `state-files-guide.md` updated to mandate validate-state at STEP 0.

### Backward compatibility

- No breaking change. Stale state files get a WARN on next resume → user chooses how to proceed. New skill is opt-in via STEP 0; skills with the new sub-step no-op when STATE.md is absent; the hook no-ops when STATE.md is not in cwd.

---

## v5.7.0 (2026-05-21)

### Fast Lane skill + upstream sync

- **`fast-lane-assessment-v1` skill (NEW)** — fills the gate that STEP 1 of the workflow named but never had a runnable skill for. Scores five criteria + hard exclusions → emits JSON → user confirmation. References existing sources of truth (criteria in `current-process-workflow.md` STEP 1, output schema in `agent-output-templates.md` T6) instead of duplicating them.
- **Upstream sync (fork-block preserved)** — `using-git-worktrees` (Step 0 isolation detection, native-tool preference with git fallback), `finishing-a-development-branch` (environment detection, detached-HEAD menu, CWD safety, provenance-based cleanup), and `subagent-driven-development` (upstream "Continuous execution" paragraph, with the fork's Pre-Task Confirmation preserved and an explicit `mode: yolo` exception).
- **Diff audit** — seven skills confirmed byte-identical to upstream; four intentionally retained fork-ahead (documented).

### Backward compatibility

- No config schema changes. No breaking changes to Mode A or Mode B; STEP 1 now resolves to a real skill instead of a missing reference. Worktree changes are additive.

---

## v5.6.0 (2026-05-07)

### Mode B Claude-only Policy

Mode B (Team Spine) is now officially **Claude Code only**. On Cursor / Codex / OpenCode, Mode Selection Gate auto-forces Mode A regardless of task complexity. Rationale: Mode B requires Claude-specific Task tool features that don't translate to other harnesses.

Mode A continues to work cross-platform on all 4 harnesses (Claude Code, Cursor, Codex, OpenCode).

### Detection

Mode Gate detects active harness via:
1. `.planning/config.json` field `active_harness` (explicit user override; default `auto`)
2. Environment variables (`CLAUDECODE`, `CURSOR_*`, `CODEX_*`, `OPENCODE_*`)
3. Fallback `claude-code` (preserves current behavior in ambiguous cases)

If detected != `claude-code` → Mode Gate forces Mode A. User can override per existing user-decision-wins rule, with hard warning shown.

### New artifacts

- **`docs/claude/harness-compatibility.md`** — full compatibility matrix + rationale + workaround guidance.
- **`active_harness` config field** added to schema + template.

### Backward compatibility

100% — v5.5.x projects continue to work. `active_harness` auto-resolves via env detection.

---

## v5.5.1 (2026-05-07)

### Cross-Platform Fixes

- **Plugin path detection** — `/init-project`, `/update-config`, `/add-tech-stack`, `/sync-stack-skill` now work on Codex / Cursor / OpenCode in addition to Claude Code. Skill bodies try 4 candidate dirs in priority order with multi-harness Glob fallback.
- **Codex INSTALL.md** — adds `agents/` symlink for Mode B users + note that Codex doesn't register slash commands (invoke skills by name).
- **Cursor plugin.json** — keyword drift fixed (synced with Claude plugin.json: +4 keywords).

### Patch nature

This is a non-breaking patch on top of v5.5.0. No new features, no API changes, no schema changes. Existing v5.5.0 customizations and snapshots continue to work unchanged.

---

## v5.5.0 (2026-05-07)

### Stack Customization Commands

- **`/add-tech-stack`** — customize an existing plugin stack skill for the current project, or add a brand-new stack (Python, Go, Rust, etc.). Two flows:
  - **Scenario A (existing stack)** — registered already (default 5 or previously added). Walk each section with `[keep / override / append / skip]`. User picks file scope: SKILL only / AGENT only / both.
  - **Scenario B (new stack)** — not registered. Forces both SKILL.md + AGENT.md. Walk template sections with `[input / suggest / skip]`. Best-practice MINIMUM filled on explicit skip. Variable section loop for stack-specific patterns.
- **`/sync-stack-skill`** — three-way merge a customized snapshot with the latest plugin default. Per-section conflict detection (4 patterns: both unchanged / plugin-only / user-only / both updated); manual-merge sub-flow with 3-pane view. Supports `--all` for batch sync.

### Architecture Detection (T1)

- **Opt-in heuristic detection** integrated into `/init-project` brownfield flow — scans folder structure + 2-3 representative files to propose project's actual architecture (Clean Architecture / Vertical Slices / N-Tier / Feature-folders / Atomic Design). Confidence levels (HIGH/MEDIUM/LOW). Web search opt-in. E3 conservative — every step requires explicit user consent.

### Snapshot Model + Stack Registry

- **`.claude/stack-skills/registry.json`** — single source of truth for stacks registered in a project. Default 5 plugin stacks + user-added stacks. Entry tracks `source` (`plugin`/`customized`/`project`), paths, sync history.
- **Project snapshot wins** — when `/add-tech-stack` flips source `plugin → customized`, registry paths rewrite to `.claude/...` snapshot files. Plugin defaults remain available unchanged at plugin install location.
- **Three new templates** at `docs/claude/templates/`: `registry.json`, `stack-skill-template.md`, `stack-agent-template.md`.

### Documentation Updates

- **`docs/claude/templates/CLAUDE.md`** — Stack-Specific Rules table now wrapped in `<!-- stack-table:start --> ... <!-- stack-table:end -->` markers; auto-regenerated by `/add-tech-stack` and `/sync-stack-skill` from registry.
- **`docs/claude/stack-skill-rule-map.md`** — runtime lookup order documented; T2 unknown stack mid-task halt behavior documented (no auto-create — user must explicitly invoke `/add-tech-stack`).

### Backward Compatibility

- v5.4.0 projects continue to work unchanged. Stack registry is auto-created at first `/add-tech-stack` or `/sync-stack-skill` invocation if missing. CLAUDE.md without markers prompts user to refresh template.

---

## v5.4.0 (2026-05-06)

### Bootstrap Commands

- **`/init-project`** — bootstrap project state files, config, CLAUDE.md, permissions, and gitignore in one command. Asks 2 minimal prompts (project name + primary stack), supports `--force` flag for re-init with backup, brownfield auto-detection via Glob markers (`*.csproj`, `package.json`, etc.). Idempotency: 4-choice menu when STATE.md exists. Cross-platform plugin path detection (Windows + macOS/Linux) with Glob fallback for non-marketplace installs.
- **`/update-config`** — interactive config update with field validation against schema, mid-workflow impact warnings, summary diff before apply. Auto-updates `.gitignore` when `commit_docs.planning_artifacts` toggles. Trigger phrases: "update config", "change config", "change model", "change commit strategy".

### Single Source of Truth

- **`docs/claude/config-schema.md`** — new schema doc consolidates all 9 config fields (12 rendered rows for nested fields). Documents valid values, defaults, mid-workflow impact, display grouping convention, and Read-permission trade-off. Both new skills defer to schema at runtime instead of hardcoding values.

### Project Templates

- **`docs/claude/templates/CLAUDE.md`** — project-level CLAUDE.md template (consumed by `/init-project`). Generic, no plugin-specific naming, embedded config schema, full STEP 7 operational details, Stack-Specific Rules table, Project-Specific Deviations table.
- **`docs/claude/templates/settings.json`** — Option C workflow-aware permission allowlist (read-only ops pre-allowed, writes/destructive require user approval).
- **`docs/claude/templates/gitignore`** — base entries for new projects.

### Fixed

- **Marketplace version drift** — `.claude-plugin/marketplace.json` was lagging at 5.2.0 (vs other manifests at 5.3.0). All 6 manifests now synced at 5.4.0.

---

## v5.3.0 (2026-04-13)

### Per-Task Model Config & Granular Commit Control

- **Per-task model assignment** — each plan task now has a `<model>` element (`haiku`/`sonnet`/`opus`). Planner assigns from `model_defaults` in config. User can override before execution.
- **Granular commit control** — `commit_docs` changed from boolean to `{state_files: bool, planning_artifacts: bool}`. `commit_atomic` controls per-task vs per-wave commits.
- **Config management** — new "update config" anytime mechanism. Resume sessions display config with keep/edit prompt. Mid-workflow config changes warn about impact on completed steps.
- **Pre-task confirmation** — Step 7 now shows model + files before each task, asks if user wants to switch model. Commit strategy confirmed before first wave.
- **Cleaned up config** — removed redundant fields (verifier, nyquist_validation, auto_advance). All workflow docs translated to English.

### Version Control & README

- **Single source of truth** — `package.json` is the canonical version. All references (`marketplace.json`, badges, RELEASE-NOTES, CHANGELOG) sync from it.
- **README rewrite** — 17-section structure with TOC, badges, project structure, configuration guide, prerequisites, troubleshooting/FAQ, contributing link, and attribution.
- **CHANGELOG backfill** — added entries for v5.1.0 and v5.2.0 from RELEASE-NOTES.

### Version Skill

- **`/version` command** — new skill that reads `package.json` and displays the current plugin version. Quick way to verify which version is installed.

---

## v5.2.0 (2026-04-08)

### GSD + Superpowers Unified Workflow

Complete integration of GSD (Get Shit Done) workflow mechanics into the Superpowers plugin.

- **11-step unified workflow** — combined GSD's state persistence, research phase, wave-based execution, and UAT gate with Superpowers' Mode A/B, QA gate, and DevOps phase
- **State files** — PROJECT.md, STATE.md, REQUIREMENTS.md, ROADMAP.md, `.planning/config.json` persist across sessions — no more memory loss between sessions
- **Research phase** — Mode A (2 agents) / Mode B (4 agents + Synthesizer) with [VERIFIED/CITED/ASSUMED] claim provenance tags
- **Goal-backward planning** — Observable Truths → Required Artifacts → Required Wiring → must_haves frontmatter before writing tasks
- **Nyquist Rule** — all automated verify steps must complete in <60s
- **Plan Checker** — 11-dimension validation before execution
- **Wave execution** — intra-wave file overlap check, fresh context per task, atomic commits, worktree isolation
- **10 human touchpoints** — AI never auto-advances between steps without user confirmation

### Implementer Skills Rewritten (2026 Best Practices)

All 5 implementer skills rewritten with syntax examples, decision trees, and anti-pattern tables:

- **React/TypeScript** — TanStack Query v6, Zustand, React Hook Form + Zod, discriminated unions, compound components
- **Angular/TypeScript** — Signals (signal/computed/effect, input()/output()), RxJS with takeUntilDestroyed, OnPush mandatory
- **IoT Edge** — MQTT QoS selection table, topic registry, LWT, exponential backoff + jitter, BLE GATT profile, schema versioning, TLS/mTLS
- **React Native/TypeScript** — RootStackParamList typed navigation, TanStack Query networkMode offlineFirst, MMKV, FlatList full optimization, CodePush ON_NEXT_RESTART

### Agent Updates

All 20 agents aligned with unified workflow:

- Implementer agents now reference their corresponding skill (REQUIRED directive)
- Phase lead agents updated with REQ-ID format, goal-backward methodology, STATE.md updates
- team-orchestrator enforces no-auto-advance rule at every phase boundary
- master-dispatcher includes research routing and UAT template routing

### Plugin Manifest

- Renamed plugin: `superpowers-ai-team-personal` v5.2.0
- All platform configs updated: Claude Code, Cursor, Codex, OpenCode, Gemini
- marketplace.json added to repo root for `/plugin marketplace add` support
- Repo: [Nhanddtse61874/happypowerprocess](https://github.com/Nhanddtse61874/happypowerprocess)

---

## v5.1.0 (2026-04-06)

### Initial Personalized Fork

- Forked from obra/superpowers
- Added AI Team orchestration agents (phase leads, orchestrator, dispatcher)
- Added GSD workflow documentation stubs
- Initial CLAUDE.md with workspace-specific execution rules
