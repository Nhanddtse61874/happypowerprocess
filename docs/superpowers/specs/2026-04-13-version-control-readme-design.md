# Version Control & README Improvement — Design Spec

**Date:** 2026-04-13
**Version target:** 5.3.0
**Scope:** Sync all version references to 5.3.0, rewrite README.md with comprehensive structure, update CHANGELOG.md

---

## 1. Problem Statement

Two issues with the current repo:

1. **Version inconsistency** — `package.json` says `5.0.7`, `marketplace.json` says `5.2.0`, `CHANGELOG.md` stops at `5.0.5`. No single source of truth.
2. **README gaps** — Missing version badge, TOC, prerequisites, project structure, configuration guide, troubleshooting, contributing link, and attribution. Content is accurate but hard to navigate and lacks detail for new users.

---

## 2. Version Control

### Source of truth

`package.json` is the single source of truth for version. All other files reference or mirror this value.

### Files to update

| File | Current | Target | Action |
|---|---|---|---|
| `package.json` | `5.0.7` | `5.3.0` | Update `version` field |
| `marketplace.json` | `5.2.0` | `5.3.0` | Update `plugins[0].version` field |
| `RELEASE-NOTES.md` | latest `5.2.0` | Add `5.3.0` section | Prepend new section at top |
| `CHANGELOG.md` | stops at `5.0.5` | Add `5.1.0`, `5.2.0`, `5.3.0` | Backfill from RELEASE-NOTES, add new entry |

### RELEASE-NOTES.md — new 5.3.0 section

```markdown
## v5.3.0 (2026-04-13)

### Version Control & README

- Established `package.json` as the single source of truth for version
- Synced all version references (`marketplace.json`, RELEASE-NOTES, CHANGELOG)
- Rewrote README with comprehensive structure: TOC, badges, project structure, configuration guide, troubleshooting, contributing, attribution
- Backfilled CHANGELOG.md with 5.1.0 and 5.2.0 entries
```

### CHANGELOG.md — backfill strategy

Prepend entries for `5.1.0`, `5.2.0`, and `5.3.0` above the existing `5.0.5` entry. Content sourced from `RELEASE-NOTES.md`. Format follows Keep a Changelog convention (Added / Changed / Fixed).

---

## 3. README.md — New Structure

### Design principles

1. **Complete detail** — every feature and component is described
2. **Clear usage** — explain how to use each part, not just what it is
3. **Short, clear sentences** — no jargon walls, no filler

### Section-by-section spec

#### 3.1 Header + Badges

```markdown
# Superpowers — Personalized Plugin

> AI workflow plugin: structured brainstorming, wave-based execution,
> persistent state, and 9 human checkpoints — all in your coding agent.

![Version](https://img.shields.io/badge/version-5.3.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Platform](https://img.shields.io/badge/platform-Claude%20Code%20|%20Codex%20|%20Cursor%20|%20OpenCode-purple)
```

Badges use shields.io static badges (no external service dependency).

#### 3.2 Table of Contents

Markdown anchor links to all 17 sections. Helps users jump to what they need.

#### 3.3 What Is This?

4 short paragraphs:
- **Line 1:** What it is — a personalized plugin for coding agents that adds structured workflow
- **Line 2:** The problem — context rot (quality drops as context fills) and memory loss (new session = blank slate)
- **Line 3:** The solution — 11-step workflow with state files that persist across sessions, fresh context per task
- **Line 4:** Origin — forked from `obra/superpowers`, extended with GSD state management and AI Team orchestration

#### 3.4 How It Works

Keep the existing 11-step table. Add introductory sentence: "The AI does not auto-advance. Every step requires your approval."

No changes to the step descriptions — they are already clear and concise.

#### 3.5 Runtime Modes

Two subsections (Mode A, Mode B), each with:
- **When to use:** 1 sentence
- **What runs:** bullet list of skills/agents involved (3-5 items)
- **Link:** to detailed docs (`docs/claude/runtime-modes.md`, `docs/claude/mode-selection-criteria.md`)

Shorter than current README — remove inline skill lists, replace with summary + link.

#### 3.6 Skills Library

Three tables: Process Skills, Execution Skills, Quality Skills.

Each row has:
| Skill | Description | Activates when |
|---|---|---|

- **Description column** is new — 1 sentence explaining what the skill does (not just when it triggers)
- Keeps all 19 skills currently listed

#### 3.7 Implementer Skills

Keep existing table (5 rows: .NET, React, Angular, React Native, IoT Edge).

Add introductory sentence: "Every implementation task must load the skill matching its stack. This is enforced, not optional."

#### 3.8 Project Structure

Tree view of repo root with 1-line description per directory/file:

```
happypowerprocess/
├── skills/                — 19 workflow skills (process, execution, quality, implementer)
├── agents/                — 12 AI Team agents (phase leads, implementers, orchestrator)
├── docs/
│   ├── claude/            — Workflow docs, templates, guides
│   ├── plans/             — Planning artifacts
│   └── superpowers/       — Specs and design docs
├── tests/                 — Test suites
├── .github/               — Issue templates, PR template
├── CLAUDE.md              — Contributor guidelines + workspace execution rules
├── CHANGELOG.md           — Full version history (all releases)
├── RELEASE-NOTES.md       — Latest release highlights
├── package.json           — Version source of truth
└── marketplace.json       — Plugin marketplace metadata
```

#### 3.9 State Files

Keep existing tree view showing the state file layout in a user's project.

Add a table explaining each file:

| File | Purpose | When to update |
|---|---|---|
| `PROJECT.md` | Vision and goals | Rarely changes |
| `REQUIREMENTS.md` | REQ-IDs with acceptance criteria | After brainstorm (Step 2) |
| `ROADMAP.md` | Milestones and phase progress | After each completed phase |
| `STATE.md` | Current step, next action, blockers, decisions | Every step transition |
| `.planning/config.json` | Mode, commit, model settings | Step 0 or via "update config" |

#### 3.10 Planning Quality Gates

Three subsections, each 3-5 lines max:

1. **Research (Step 4)** — claim provenance tags: VERIFIED, CITED, ASSUMED
2. **Plan (Step 6)** — goal-backward methodology, task structure (read_first, action, verify, done), Nyquist rule
3. **Execution (Step 7)** — wave-based, fresh context, atomic commits, file overlap check

Shorter than current README. Detail lives in `docs/claude/`.

#### 3.11 Prerequisites

```
- Git
- Node.js >= 18 (required for brainstorm server)
- One of: Claude Code, Codex, Cursor, or OpenCode
```

Short explanation of why each is needed.

#### 3.12 Installation

Keep all existing platform sections (Claude Code marketplace, Claude Code local, Codex, OpenCode, Cursor).

Changes:
- Add "See [Prerequisites](#prerequisites)" at top
- Expand "Verify Installation" — show expected behavior (agent should invoke brainstorming skill)
- Keep "Update" section as-is

#### 3.13 Configuration

Explain `.planning/config.json` with a table:

| Key | Description | Values |
|---|---|---|
| `mode` | Solo or AI Team | `"A"` / `"B"` |
| `granularity` | Plan detail level | `"low"` / `"medium"` / `"high"` |
| `parallelization` | Run tasks in parallel | `true` / `false` |
| `commit_atomic` | Commit per task or per wave | `true` (per task) / `false` (per wave) |
| `commit_docs` | Which docs to commit | `{state_files: bool, planning_artifacts: bool}` |
| `model_defaults` | Model per task complexity | `{mechanical: "haiku", standard: "sonnet", complex: "opus"}` |

Include a sample config JSON block.

How to update: say "update config" at any time during a session.

#### 3.14 Troubleshooting / FAQ

| Problem | Solution |
|---|---|
| Brainstorm server won't start | Check Node.js >= 18. See CHANGELOG 5.0.5 ESM fix. |
| Plugin doesn't load skills | Re-run installation steps. Check symlinks (Codex). |
| State files lost after branch switch | State files live in working directory, not git branch. |
| Want to change mode after selection | Run "update config" or restart from Step 3. |

#### 3.15 Contributing

Short section:
- Link to `CLAUDE.md` for full contributor guidelines
- Link to `.github/PULL_REQUEST_TEMPLATE.md`
- 3 key rules: read PR template fully, search for duplicate PRs, one problem per PR

#### 3.16 Attribution

```
Forked from [obra/superpowers](https://github.com/obra/superpowers) by Jesse Vincent.
Original license: MIT.
```

#### 3.17 License

```
MIT License — see [LICENSE](LICENSE) for details.
```

---

## 4. RELEASE-NOTES v5.3.0 Content

```markdown
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
```

---

## 5. Version Skill

### Purpose

Let users check the installed plugin version by typing `/version` in any session.

### Behavior

1. Read `package.json` from the plugin root directory
2. Extract the `version` field
3. Display: plugin name, version number, and repo URL

### File location

`skills/version/SKILL.md`

### Skill content (draft)

```markdown
---
name: version
description: Display the current plugin version
---

Read the `package.json` file in the plugin root directory. Extract the `version` field. Display:

**Superpowers Plugin** — v{version}
Repository: https://github.com/Nhanddtse61874/happypowerprocess
```

Simple, no side effects, read-only.

---

## 6. Out of Scope

- No changes to existing skills, agents, or workflow docs
- No changes to CLAUDE.md
- No changes to platform-specific install docs (`.codex/`, `.opencode/`, etc.)

---

## 7. Acceptance Criteria

- [ ] `package.json` version is `5.3.0`
- [ ] `marketplace.json` version is `5.3.0`
- [ ] `CHANGELOG.md` has entries for `5.1.0`, `5.2.0`, `5.3.0`
- [ ] `RELEASE-NOTES.md` has `5.3.0` section at top
- [ ] `README.md` has all 17 sections as specified
- [ ] All version badges in README show `5.3.0`
- [ ] README TOC links work (anchor links match headings)
- [ ] No broken markdown formatting
- [ ] `skills/version/SKILL.md` exists and works when invoked via `/version`
