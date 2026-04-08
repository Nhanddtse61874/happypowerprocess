# Superpowers — Personalized Plugin

## Overview

This workspace is a personalized plugin that combines:
- **Superpowers** as the execution skills engine
- **GSD (Get Shit Done)** state management, research phase, wave-based execution, UAT gate, and milestone rhythm
- **Optional AI Team** orchestration as the structural backbone

The result is an **11-step workflow** that gives your coding agent persistent memory across sessions, evidence-based planning, and explicit human checkpoints at every key decision — without losing any of the discipline from Superpowers.

---

## How It Works

### The Problem It Solves

Two problems kill long AI coding sessions:
1. **Context rot** — quality degrades as the context window fills up
2. **Memory loss** — every new session starts from scratch

This workflow solves both: state files persist across sessions, and every implementation task runs in a fresh isolated context.

### The 11-Step Flow

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

**Principle:** The AI accompanies you through each step — it never auto-advances without your approval.

---

## Runtime Modes

### Mode A — Solo (Superpowers)

For single-domain tasks, lower complexity, no formal QA gate needed.

- Brainstorming skill writes spec solo
- writing-plans skill creates plan
- subagent-driven-development or executing-plans runs implementation
- requesting-code-review skill runs QA
- finishing-a-development-branch closes the branch

### Mode B — AI Team Spine

For multi-domain tasks, high-risk changes, or when you need formal QA/DevOps gates.

- `phase-discovery-lead` formalizes requirements spec
- `phase-architecture-lead` formalizes technical spec
- `phase-implementation-lead` writes implementation plan
- Stack implementer agents execute (React Native, .NET/C#, Angular, React, IoT Edge)
- `phase-qa-gate` + `qa-code-reviewer` run QA with JSON output contracts
- `phase-release-devops-lead` handles CI/CD and release readiness
- `team-orchestrator` coordinates and summarizes at phase boundaries

**Mode is selected after Step 2 (Brainstorm) using 5 scored criteria. User approves — mode does not change after that.**

---

## Project State Files

State files give your AI persistent memory that survives context resets and new sessions.

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

Templates for all files: `docs/claude/templates/`

---

## Planning Quality Gates

### Research Phase (Step 4)

Every claim in `RESEARCH.md` is tagged:
- `[VERIFIED: source]` — tool-confirmed
- `[CITED: url]` — from official documentation
- `[ASSUMED]` — training knowledge, needs validation

### Plan Structure (Step 6)

Plans use goal-backward methodology: Observable Truths → Required Artifacts → Required Wiring → Key Links → `must_haves` frontmatter.

Each task requires:
- `<read_first>` — files to read before starting
- `<action>` — specific implementation with what to avoid and why
- `<verify><automated>` — runnable command completing in <60 seconds (Nyquist Rule)
- `<done>` — grep-verifiable acceptance criteria

**Plan Checker** validates 11 dimensions before execution (requirement coverage, Nyquist compliance, scope sanity, dependency correctness, and more).

### Execution (Step 7)

- **Wave-based**: independent tasks run in parallel, dependent tasks wait
- **Fresh context per task**: each subagent receives only the files it needs
- **Atomic commits**: every task commits immediately on completion
- **Intra-wave file overlap check**: tasks modifying the same file are forced sequential

---

## Implementer Skills

Stack-specific skills enforced on every implementation task:

| Stack | Skill | Key Coverage |
|---|---|---|
| .NET / C# | `implementer-dotnet-csharp` | Clean Architecture, SOLID, FluentValidation, async/await, global exception middleware |
| React / TypeScript | `implementer-react-typescript` | TanStack Query, Zustand, React Hook Form + Zod, discriminated unions, compound components |
| Angular / TypeScript | `implementer-angular-typescript` | Signals (Angular 17+), RxJS operators, takeUntilDestroyed, OnPush, HTTP interceptors |
| React Native / TypeScript | `implementer-react-native-typescript` | Typed navigation, FlatList optimization, MMKV, CodePush safety, platform differences |
| IoT Edge (MQTT/BLE) | `implementer-iot-edge` | QoS selection, topic registry, LWT, exponential backoff, BLE lifecycle, schema versioning, TLS |

---

## Skills Library

### Process Skills

| Skill | When it activates |
|---|---|
| `using-superpowers` | Start of every session |
| `brainstorming` | Before any implementation — explores requirements, proposes approaches |
| `writing-plans` | After spec approval — creates bite-sized implementation plan |
| `writing-skills` | When creating or editing skills |

### Execution Skills

| Skill | When it activates |
|---|---|
| `subagent-driven-development` | Plan execution — fresh subagent per task with review |
| `executing-plans` | Plan execution — inline batch execution with checkpoints |
| `dispatching-parallel-agents` | Independent tasks that can run concurrently |
| `using-git-worktrees` | Before feature work needing isolation |
| `finishing-a-development-branch` | When implementation is complete |

### Quality Skills

| Skill | When it activates |
|---|---|
| `test-driven-development` | During implementation — RED→GREEN→REFACTOR |
| `requesting-code-review` | After implementation — severity-classified findings |
| `receiving-code-review` | When processing review feedback |
| `systematic-debugging` | When a bug or unexpected behavior is encountered |
| `verification-before-completion` | Before claiming work is done |

---

## Key Principles

- **Human in the loop** — 9 explicit touchpoints, AI waits for approval at each
- **Evidence over assumption** — research phase with claim provenance before planning
- **Context budget discipline** — 2-3 tasks per plan, fresh context per task, no compression
- **Atomic progress** — every completed task is a committed, revertable git commit
- **State persistence** — STATE.md is the source of truth, survives any context reset
- **Test-Driven Development** — RED→GREEN→REFACTOR, always
- **YAGNI + DRY** — no speculative features, no duplication

---

## Primary Reference Docs

| Doc | Purpose |
|---|---|
| `docs/claude/current-process-workflow.md` | Full 11-step workflow with all rules |
| `docs/claude/state-files-guide.md` | State files — what they are, when to update |
| `docs/claude/research-phase-guide.md` | Research phase agents, claim provenance |
| `docs/claude/mode-selection-criteria.md` | How to score and select Mode A vs B |
| `docs/claude/agent-output-templates.md` | Output template contracts (JSON) |
| `docs/claude/stack-skill-rule-map.md` | Mandatory skill per stack |
| `docs/claude/workflow-diagram.md` | Mermaid diagram of full flow |
| `docs/claude/templates/` | Templates for all state and phase output files |

---

## Installation

> **Prerequisite:** Repo must be **public** on GitHub for marketplace install to work.

### Claude Code — Marketplace (recommended)

```bash
# Step 1: Add this repo as a marketplace
/plugin marketplace add Nhanddtse61874/privateAgentTeam

# Step 2: Install the plugin
/plugin install superpowers-ai-team-personal@superpowers-ai-team-personal-dev
```

### Claude Code — Local (no public repo needed)

```bash
# Clone the repo
git clone https://github.com/Nhanddtse61874/privateAgentTeam

# Load plugin for the session
claude --plugin-dir ./privateAgentTeam
```

### Verify Installation

Start a new session and say something like _"let's build a feature"_. The agent should automatically invoke the `brainstorming` skill.

### Reload after update

```bash
/reload-plugins
```

---

## License

MIT License — see LICENSE file for details.
