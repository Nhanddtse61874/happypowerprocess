# Release Notes — Superpowers Personalized Plugin

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
- **9 human touchpoints** — AI never auto-advances between steps without user confirmation

### Implementer Skills Rewritten (2026 Best Practices)

All 4 implementer skills rewritten with syntax examples, decision trees, and anti-pattern tables:

- **React/TypeScript** — TanStack Query v6, Zustand, React Hook Form + Zod, discriminated unions, compound components
- **Angular/TypeScript** — Signals (signal/computed/effect, input()/output()), RxJS with takeUntilDestroyed, OnPush mandatory
- **IoT Edge** — MQTT QoS selection table, topic registry, LWT, exponential backoff + jitter, BLE GATT profile, schema versioning, TLS/mTLS
- **React Native/TypeScript** — RootStackParamList typed navigation, TanStack Query networkMode offlineFirst, MMKV, FlatList full optimization, CodePush ON_NEXT_RESTART

### Agent Updates

All 12 agents aligned with unified workflow:

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
