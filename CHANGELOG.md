# Changelog

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
