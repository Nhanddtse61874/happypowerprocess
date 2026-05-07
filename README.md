# happypowerprocess — Personalized Plugin

> AI workflow plugin: structured brainstorming, wave-based execution, persistent state, and 10 human checkpoints — all in your coding agent.

![Version](https://img.shields.io/badge/version-5.6.1-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Platform](https://img.shields.io/badge/platform-Claude%20Code%20|%20Codex%20|%20Cursor%20|%20OpenCode-purple)

---

## Table of Contents

- [What Is This?](#what-is-this)
- [How It Works](#how-it-works)
- [Runtime Modes](#runtime-modes)
- [Skills Library](#skills-library)
- [Implementer Skills](#implementer-skills)
- [Project Structure](#project-structure)
- [State Files](#state-files)
- [Planning Quality Gates](#planning-quality-gates)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Troubleshooting / FAQ](#troubleshooting--faq)
- [Primary Reference Docs](#primary-reference-docs)
- [Contributing](#contributing)
- [Attribution](#attribution)
- [License](#license)

---

## What Is This?

happypowerprocess is a personalized plugin for coding agents. It adds a structured 11-step workflow to your AI coding sessions — from brainstorming through shipping.

Two problems kill long AI coding sessions: **context rot** (quality drops as the context window fills up) and **memory loss** (every new session starts from scratch). This plugin solves both.

The workflow uses **state files** that persist across sessions, so your AI always knows where you left off. Every implementation task runs in a **fresh isolated context**, so quality stays high even in long sessions.

This project is forked from [obra/superpowers](https://github.com/obra/superpowers). It extends the original with **GSD (Get Shit Done)** state management, a research phase, wave-based execution, and optional **AI Team** orchestration with specialized agents.

---

## How It Works

The AI does not auto-advance. Every step requires your approval.

```
STEP 0:  Bootstrap          → Read STATE.md (resume) or create project state files
STEP 1:  Fast Lane?         → Hotfix/small task: skip to STEP 5
STEP 2:  Brainstorm         → Understand requirements, explore approaches     [USER APPROVES]
STEP 3:  Mode Gate          → Mode A (solo) or Mode B (AI team)               [USER APPROVES]
STEP 4:  Research           → Parallel research agents — evidence before plan  [USER REVIEWS]
STEP 5:  Spec               → Write technical spec from brainstorm + research  [USER APPROVES]
STEP 6:  Plan               → Goal-backward XML tasks, wave-grouped            [USER APPROVES]
STEP 7:  Execute            → Wave by wave, fresh context per task, atomic commits [USER APPROVES each wave]
STEP 8:  UAT + Verify       → User tests feature + goal-backward verification  [USER DRIVES]
STEP 9:  QA Gate            → Severity-classified findings, fix loop           [USER APPROVES]
STEP 10: Release / DevOps   → CI/CD plan, rollback, observability (Mode B)    [USER APPROVES]
STEP 11: Ship               → PR/merge + SUMMARY + ROADMAP + STATE updated    [USER DECIDES]
```

There are **10 explicit human checkpoints**. The AI accompanies you through each step — it never moves forward without your say-so.

---

## Runtime Modes

After brainstorming (Step 2), the workflow scores 5 criteria and suggests a mode. You approve or override. The mode does not change after that.

### Mode A — Solo (Superpowers)

Use for single-domain tasks with lower complexity. No formal QA gate needed.

- `brainstorming` writes the spec
- `writing-plans` creates the implementation plan
- `subagent-driven-development` or `executing-plans` runs implementation
- `requesting-code-review` runs QA
- `finishing-a-development-branch` closes the branch

### Mode B — AI Team

Use for multi-domain tasks, high-risk changes, or when you need formal QA/DevOps gates.

- `phase-discovery-lead` formalizes the requirements spec
- `phase-architecture-lead` formalizes the technical spec
- `phase-implementation-lead` writes the implementation plan
- Stack-specific implementer agents execute tasks
- `phase-qa-gate` + `qa-code-reviewer` run QA with JSON output contracts
- `phase-release-devops-lead` handles CI/CD and release readiness
- `team-orchestrator` coordinates at phase boundaries

See [runtime-modes.md](docs/claude/runtime-modes.md) and [mode-selection-criteria.md](docs/claude/mode-selection-criteria.md) for details.

---

## Skills Library

Skills are specialized behaviors the agent loads on demand. They activate automatically when relevant, or you can invoke them with `/skill-name`.

### Process Skills

| Skill | Description | Activates when |
|---|---|---|
| `using-superpowers` | Loads skill discovery and routing at session start | Start of every session |
| `brainstorming` | Explores requirements, proposes 2-3 approaches, writes design spec | Before any implementation |
| `writing-plans` | Creates bite-sized XML tasks grouped into waves | After spec approval |
| `writing-skills` | Guides creation and testing of new skills | Creating or editing skills |

### Execution Skills

| Skill | Description | Activates when |
|---|---|---|
| `subagent-driven-development` | Dispatches a fresh subagent per task with two-stage review | Plan execution (recommended) |
| `executing-plans` | Runs tasks inline with batch execution and checkpoints | Plan execution (alternative) |
| `dispatching-parallel-agents` | Runs independent tasks concurrently across multiple agents | 2+ tasks with no shared state |
| `using-git-worktrees` | Creates isolated git worktrees for feature work | Before feature work needing isolation |
| `finishing-a-development-branch` | Guides merge, PR, or cleanup when implementation is complete | After all tasks pass |

### Quality Skills

| Skill | Description | Activates when |
|---|---|---|
| `test-driven-development` | Enforces RED → GREEN → REFACTOR cycle | During implementation |
| `requesting-code-review` | Produces severity-classified findings with fix suggestions | After implementation |
| `receiving-code-review` | Processes review feedback with technical rigor before implementing | When review feedback arrives |
| `systematic-debugging` | Structured root-cause analysis before proposing fixes | When a bug or unexpected behavior appears |
| `verification-before-completion` | Runs verification commands and confirms output before claiming done | Before claiming work is complete |

### Utility Skills

| Skill | Description | Activates when |
|---|---|---|
| `version` | Displays the current plugin version from `package.json` | User types `/version` |

---

## Implementer Skills

Every implementation task must load the skill matching its stack. This is enforced, not optional.

| Stack | Skill | Key Coverage |
|---|---|---|
| .NET / C# | `implementer-dotnet-csharp` | Clean Architecture, SOLID, FluentValidation, async/await, global exception middleware |
| React / TypeScript | `implementer-react-typescript` | TanStack Query, Zustand, React Hook Form + Zod, discriminated unions, compound components |
| Angular / TypeScript | `implementer-angular-typescript` | Signals (Angular 17+), RxJS operators, takeUntilDestroyed, OnPush, HTTP interceptors |
| React Native / TypeScript | `implementer-react-native-typescript` | Typed navigation, FlatList optimization, MMKV, CodePush safety, platform differences |
| IoT Edge (MQTT/BLE) | `implementer-iot-edge` | QoS selection, topic registry, LWT, exponential backoff, BLE lifecycle, schema versioning, TLS |

---

## Project Structure

```
happypowerprocess/
├── skills/                    — 20 workflow skills (process, execution, quality, implementer, utility)
│   ├── brainstorming/         — Design exploration and spec writing
│   ├── writing-plans/         — Implementation plan creation
│   ├── subagent-driven-development/ — Fresh subagent per task execution
│   ├── test-driven-development/     — TDD enforcement
│   ├── implementer-react-typescript/ — React/TS implementation rules
│   ├── version/               — Plugin version display
│   └── ...                    — 14 more skills
├── agents/                    — 20 AI Team agents
│   ├── phase-discovery-lead.md      — Requirements formalization
│   ├── phase-architecture-lead.md   — Technical spec
│   ├── phase-implementation-lead.md — Plan writing
│   ├── phase-qa-gate.md             — QA gate with JSON contracts
│   ├── phase-release-devops-lead.md — CI/CD and release readiness
│   ├── team-orchestrator.md         — Cross-phase coordination
│   ├── master-dispatcher.md         — Task routing
│   └── ...                          — 13 more agents (implementers, reviewers, architects)
├── docs/
│   ├── claude/                — Workflow docs, templates, and guides
│   │   ├── current-process-workflow.md — Full 11-step workflow reference
│   │   ├── state-files-guide.md       — State file usage guide
│   │   ├── research-phase-guide.md    — Research agents and claim provenance
│   │   ├── mode-selection-criteria.md — Mode A vs B scoring
│   │   ├── templates/                 — Templates for state and phase files
│   │   └── ...                        — More workflow docs
│   ├── plans/                 — Planning artifacts
│   └── superpowers/           — Specs and design docs
├── tests/                     — Test suites
├── .github/                   — Issue templates, PR template
├── CLAUDE.md                  — Contributor guidelines + workspace execution rules
├── CHANGELOG.md               — Full version history (all releases)
├── RELEASE-NOTES.md           — Latest release highlights
├── package.json               — Version source of truth (5.6.1)
└── marketplace.json           — Plugin marketplace metadata
```

---

## State Files

State files give your AI persistent memory that survives context resets and new sessions. They live in your project root.

```
{project-root}/
├── PROJECT.md          — Vision document (rarely changes)
├── REQUIREMENTS.md     — REQ-IDs, v1/v2 scope, acceptance criteria
├── ROADMAP.md          — Milestones and phase progress
├── STATE.md            — Current step, next action, blockers, decisions
└── .planning/
    ├── config.json              — Workflow settings (mode, granularity, parallelization)
    ├── {phase}-RESEARCH.md      — Research findings with [VERIFIED/CITED/ASSUMED] tags
    ├── {phase}-{N}-PLAN.md      — XML tasks with YAML frontmatter and must_haves
    ├── {phase}-UAT.md           — User acceptance test results
    ├── {phase}-VERIFICATION.md  — Goal-backward verification results
    └── {phase}-SUMMARY.md       — What was built, decisions made, lessons learned
```

| File | Purpose | When to update |
|---|---|---|
| `PROJECT.md` | Vision and goals for the project | Rarely — only when project direction changes |
| `REQUIREMENTS.md` | REQ-IDs with testable acceptance criteria | After brainstorm (Step 2) |
| `ROADMAP.md` | Milestones and phase progress tracking | After each completed phase |
| `STATE.md` | Current step, next action, blockers, decisions | Every step transition |
| `.planning/config.json` | Mode, commit strategy, model defaults | Step 0 or anytime via "update config" |

Templates for all files: [`docs/claude/templates/`](docs/claude/templates/)

---

## Planning Quality Gates

### Research (Step 4)

Every claim in research output is tagged with its provenance:
- `[VERIFIED: source]` — confirmed by running a tool or reading code
- `[CITED: url]` — from official documentation
- `[ASSUMED]` — from training knowledge, needs validation before use

### Plan Structure (Step 6)

Plans use goal-backward methodology: start from the desired end state and work backward. Each task includes `<read_first>` (files to read), `<action>` (what to implement), `<verify>` (automated check completing in <60 seconds per the Nyquist Rule), and `<done>` (grep-verifiable acceptance criteria).

The **Plan Checker** validates 11 dimensions before execution: requirement coverage, Nyquist compliance, scope sanity, dependency correctness, and more.

### Execution (Step 7)

Tasks run **wave by wave**. Independent tasks within a wave can run in parallel. Dependent tasks wait. Each task gets **fresh context** (no accumulated pollution). Every completed task produces an **atomic commit**. An **intra-wave file overlap check** forces tasks that modify the same file to run sequentially.

---

## Prerequisites

Before installing, make sure you have:

- **Git** — required for atomic commits, worktree isolation, and branch management
- **Node.js >= 18** — required for the brainstorm visualization server
- **A supported coding agent** — one of:
  - [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (recommended)
  - [Codex](https://github.com/openai/codex)
  - [Cursor](https://cursor.sh)
  - [OpenCode](https://opencode.ai)

---

## Installation

> **Note:** For marketplace install, the repo must be **public** on GitHub.

See [Prerequisites](#prerequisites) before starting.

### Claude Code — Marketplace (recommended)

```bash
# Step 1: Add this repo as a marketplace
/plugin marketplace add Nhanddtse61874/happypowerprocess

# Step 2: Install the plugin
/plugin install happypowerprocess@happypowerprocess
```

### Claude Code — Local (no public repo needed)

```bash
# Clone the repo
git clone https://github.com/Nhanddtse61874/happypowerprocess

# Load plugin for the session
claude --plugin-dir ./happypowerprocess
```

### Codex

```bash
# Clone the repo
git clone https://github.com/Nhanddtse61874/happypowerprocess.git ~/.codex/happypowerprocess

# Create the skills symlink
mkdir -p ~/.agents/skills
ln -s ~/.codex/happypowerprocess/skills ~/.agents/skills/happypowerprocess
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
cmd /c mklink /J "$env:USERPROFILE\.agents\skills\happypowerprocess" "$env:USERPROFILE\.codex\happypowerprocess\skills"
```

Restart Codex after install. Full guide: [.codex/INSTALL.md](.codex/INSTALL.md)

### OpenCode

```
Fetch and follow instructions from https://raw.githubusercontent.com/Nhanddtse61874/happypowerprocess/main/.opencode/INSTALL.md
```

### Cursor

```
/add-plugin https://github.com/Nhanddtse61874/happypowerprocess
```

### Verify Installation

Start a new session and say something like _"let's build a feature"_. The agent should automatically invoke the `brainstorming` skill. You can also type `/version` to check the installed version.

### Update

```bash
# Claude Code
/plugin update happypowerprocess

# Codex
cd ~/.codex/happypowerprocess && git pull
```

---

## Configuration

The workflow uses `.planning/config.json` to control behavior. This file is created during Step 0 (Bootstrap) and can be updated anytime by saying **"update config"**.

### Config Options

| Key | Description | Values |
|---|---|---|
| `mode` | Interaction style | `"interactive"` (confirm each step) |
| `granularity` | Number of phases in ROADMAP.md | `"coarse"` (3-5) / `"standard"` (5-8) / `"fine"` (8-12+) |
| `parallelization` | Run independent tasks in parallel | `true` / `false` |
| `commit_atomic` | When to commit changes | `true` (after each task) / `false` (batch per wave) |
| `commit_docs` | Which documentation to commit | `{state_files: bool, planning_artifacts: bool}` |
| `model_defaults` | Model assignment per task complexity | `{mechanical: "haiku", standard: "sonnet", complex: "opus"}` |
| `model_profile` | Shifts all model tiers up/down from defaults | `"balanced"` (no shift) / `"quality"` (tier up) / `"budget"` (tier down) |
| `workflow` | Toggle optional workflow phases | `{research: bool, plan_check: bool}` |

### Sample Config

```json
{
  "mode": "interactive",
  "granularity": "standard",
  "parallelization": true,
  "commit_docs": {
    "state_files": true,
    "planning_artifacts": true
  },
  "commit_atomic": true,
  "model_profile": "balanced",
  "model_defaults": {
    "mechanical": "haiku",
    "standard": "sonnet",
    "complex": "opus"
  },
  "workflow": {
    "research": true,
    "plan_check": true
  }
}
```

- **`commit_docs.planning_artifacts: false`** — automatically adds `.planning/` to `.gitignore`
- **`model_defaults`** — the planner assigns a model per task; you can override before execution
- **`workflow.research: false`** — skips the research phase (Step 4)
- **`workflow.plan_check: false`** — skips the 11-dimension plan validation

### How `model_profile` shifts tiers

`model_profile` applies a uniform shift to all three `model_defaults` tiers. Boundaries are clamped — `haiku` cannot shift lower, `opus` cannot shift higher.

| Profile | mechanical | standard | complex |
|---|---|---|---|
| `balanced` (default) | haiku | sonnet | opus |
| `quality` (tier up) | sonnet | opus | opus |
| `budget` (tier down) | haiku | haiku | sonnet |

Use `quality` when you want higher accuracy at higher cost. Use `budget` when you want faster, cheaper execution at lower capability.

---

## Troubleshooting / FAQ

| Problem | Solution |
|---|---|
| Brainstorm server won't start | Check that Node.js >= 18 is installed. See [CHANGELOG v5.0.5](CHANGELOG.md) for ESM fix details. |
| Plugin doesn't load skills | Re-run the installation steps for your platform. For Codex, verify the symlink points to the correct directory. |
| State files lost after branch switch | State files live in the working directory, not tied to a git branch. Switch back to the branch where you created them. |
| Want to change mode after selection | Say "update config" to modify settings. Or restart from Step 3 (Mode Gate). |
| How to check installed version | Type `/version` in any session. The agent reads `package.json` and displays the current version. |

---

## Primary Reference Docs

| Doc | Purpose |
|---|---|
| [`current-process-workflow.md`](docs/claude/current-process-workflow.md) | Full 11-step workflow with all rules |
| [`state-files-guide.md`](docs/claude/state-files-guide.md) | State files — what they are, when to update |
| [`research-phase-guide.md`](docs/claude/research-phase-guide.md) | Research phase agents, claim provenance |
| [`mode-selection-criteria.md`](docs/claude/mode-selection-criteria.md) | How to score and select Mode A vs B |
| [`agent-output-templates.md`](docs/claude/agent-output-templates.md) | Output template contracts (JSON) |
| [`stack-skill-rule-map.md`](docs/claude/stack-skill-rule-map.md) | Mandatory skill per stack |
| [`workflow-diagram.md`](docs/claude/workflow-diagram.md) | Mermaid diagram of full flow |
| [`templates/`](docs/claude/templates/) | Templates for all state and phase output files |

---

## Contributing

Read the full contributor guidelines in [CLAUDE.md](CLAUDE.md) before submitting a PR.

Key rules:
1. **Read the PR template** at [`.github/PULL_REQUEST_TEMPLATE.md`](.github/PULL_REQUEST_TEMPLATE.md) and fill in every section
2. **Search for existing PRs** (open and closed) that address the same problem
3. **One problem per PR** — do not bundle unrelated changes

This repo has a high PR rejection rate. Every rejected PR was submitted without following these guidelines. Read them carefully.

---

## Attribution

Forked from [obra/superpowers](https://github.com/obra/superpowers) by Jesse Vincent.
Original license: MIT.

---

## License

MIT License — see [LICENSE](LICENSE) for details.
