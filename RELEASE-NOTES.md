# Release Notes — happypowerprocess

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
