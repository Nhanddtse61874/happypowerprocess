# Unified Workflow: GSD + Superpowers

Combines GSD state management, research phase, wave-based execution, UAT gate, and milestone rhythm with Superpowers Mode A/B, Fast Lane, QA gate, DevOps phase, and escalation.

**Principle:** AI assists through each step — never auto-advances without human approval.

> **This file is a reference/index, not the runtime spec.** The always-loaded operational spec is `CLAUDE.md` (condensed 11 steps + rules). Each step is executed by its skill/agent (column below). Config / mode / stack reference-data lives in the SSoT files. The unique methodology this doc owns — Fast Lane criteria, goal-backward, wave algorithm, Plan Checker — is in **Reference Blocks** at the end (cited by the skills/agents that need it).

**SSoT files:** `config-schema.md` · `modes.md` · `state-files-guide.md` · `research-phase-guide.md` · `ai-team.md` · `agent-output-templates.md` · `stack-skill-rule-map.md` · `templates/`

---

## The 11 Steps

Stop after each step and wait for user confirmation — **never auto-advance**. Full per-step operational detail: `CLAUDE.md`. Mode A vs B: `modes.md`.

| Step | Purpose | Owner — Mode A / Mode B | Human touchpoint | Detail lives in |
|---|---|---|---|---|
| **0 Bootstrap** | Set up / resume project state | `validate-state` (resume, mandatory) · `init-project` (new) | confirm context + config | `config-schema.md`, `state-files-guide.md` |
| **0.5 Knowledge Bootstrap** *(optional)* | Build per-project knowledge library `.claude/skills/<project>-*` | `mining-project-knowledge` (`/mine-knowledge`) | approve inventory | Reference Blocks → Knowledge layer |
| **1 Fast Lane** | Eligible → skip steps 2-4 | `fast-lane-assessment-v1` | confirm fast vs full | Reference Blocks → Fast Lane |
| **2 Brainstorm** | Requirements, scope, 2-3 approaches, REQ-IDs | `brainstorming` | approve design direction | — |
| **3 Mode Gate** | Score 5 criteria → Mode A or B | `brainstorming` (runs gate) | approve mode (locked after) | `modes.md` |
| **4 Research** | Evidence before plan; `[VERIFIED/CITED/ASSUMED]` | 2 agents (A) / 4 + Synthesizer (B) | review sufficiency | `research-phase-guide.md` |
| **5 Spec** | Technical spec from brainstorm + research | `brainstorming` (A) / `phase-discovery-lead` + `phase-architecture-lead` (B) | approve spec | — |
| **6 Plan** | Goal-backward XML tasks, wave-grouped | `writing-plans` (A) / `phase-implementation-lead` (B) | approve plan + waves | Reference Blocks → Planning |
| **7 Execute** | Wave by wave, fresh context per task | `subagent-driven-development` / `executing-plans` | approve each wave | Reference Blocks → Execution; `config-schema.md` |
| **8 UAT + Verify** | User tests + goal-backward verification | user-driven (AI assists) | user drives | Reference Blocks → UAT; `templates/` |
| **9 QA Gate** | Severity-classified findings, fix loop | `requesting-code-review` (A) / `phase-qa-gate` + `qa-code-reviewer` (B) | approve or fix | `agent-output-templates.md` |
| **10 Release** | CI/CD, rollback, observability | `finishing-a-development-branch` (A) / `phase-release-devops-lead` (B) | approve readiness | — |
| **11 Ship** | PR/merge + SUMMARY + ROADMAP + STATE | `finishing-a-development-branch` (A) / `team-orchestrator` (B) | decide ship + next milestone | `templates/phase-SUMMARY.md` |

**Outputs (STEP 0):** `.planning/config.json`, `PROJECT.md`, `REQUIREMENTS.md` (REQ-IDs), `ROADMAP.md`, `STATE.md`.
**REQ-ID rule (STEP 2):** every v1 requirement maps to exactly one ROADMAP phase — 100% coverage; phase count within `granularity`.
**Output standard:** Mode A = Superpowers skill outputs; Mode B = `template_id` from `agent-output-templates.md`. State files updated after every step; artifacts written to disk immediately.

---

## Two Paths

```
Fast Lane:  0 → 1(eligible) → 5 → 6 → 7 → 8 → 9 → 11        (skip 2, 3, 4, 10)
Full:       0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → 11
```

Visual diagram: `workflow-diagram.md`. Config changes anytime ("update config") → `config-schema.md` (impact per field).

---

## Reference Blocks

The methodology below has **no other home** and is cited by the skills/agents that run these steps. Keep it here.

### Fast Lane eligibility (STEP 1) — cited by `fast-lane-assessment-v1`

All must be true, else use full workflow:
- Fix scope clear and unambiguous
- Small change (1-2 files)
- No architecture / API contract change
- No security or data-migration impact
- Low regression risk

Eligible → skip Steps 2-4, go to Step 5 with the assessment as input.

### Goal-Backward Planning (STEP 6)

Before writing tasks, derive from the phase goal:
1. **Observable Truths** (3-7, user perspective) — "User can send a message"
2. **Required Artifacts** (specific files) — `src/components/Chat.tsx`
3. **Required Wiring** (connections) — Chat fetches `/api/chat` on mount
4. **Key Links** (critical failure points) — where breakage cascades

→ Write into the `must_haves` frontmatter of PLAN.md.

**Task XML** (`<model>` · `<read_first>` · `<action>` incl. what-to-avoid + REQ-ID · `<verify><automated>` command <60s (Nyquist) · `<done>` grep-verifiable). Template: `templates/plan-task.xml`. Model-tier assignment (`haiku`/`sonnet`/`opus`): `config-schema.md`.
**Context budget:** ~50% tokens per plan, max 2-3 tasks; over budget → split into smaller plans, never compress.

### Wave Assignment Algorithm (STEP 6)

```
if task.depends_on is empty      → Wave 1
else                             → Wave = max(wave of dependencies) + 1
if files_modified overlap an earlier task → force later wave
```

Same-wave plans must have **zero file overlap** (required).

### Plan Checker — 11 dimensions (STEP 6, if `workflow.plan_check: true`)

Validated before approval; revision loop max 3 iterations (if issue count doesn't drop → escalate to user).

| # | Dimension | Question |
|---|---|---|
| 1 | Requirement Coverage | Every REQ-ID has a covering task? |
| 2 | Task Completeness | Every task has model + read_first + action + verify + done? |
| 3 | Dependency Correctness | Dependencies valid, acyclic, correctly waved? |
| 4 | Key Links Planned | Artifacts wired together (not just created)? |
| 5 | Scope Sanity | 2-3 tasks/plan, ~50% context? |
| 6 | Verification Derivation | must_haves trace to phase goal? |
| 7 | Context Compliance | Plans honor CONTEXT.md decisions? |
| 8 | Nyquist Compliance | Automated verify present and <60s? |
| 9 | Cross-Plan Data Contracts | Shared data transforms compatible? |
| 10 | CLAUDE.md Compliance | Plans respect project conventions? |
| 11 | Research Resolution | Open questions answered? |

**Scope Reduction Detection:** plan references user decisions but delivers "v1 stubs" → **always a blocker**.

### Execution specifics (STEP 7)

- **Fresh context per task:** each subagent gets only `<read_first>` files; controller dispatches with the task's `<model>` (or user override); escalate haiku→sonnet→opus if BLOCKED.
- **Intra-wave safety:** 2 plans modifying the same file → force sequential.
- **Commit** (`commit_atomic` semantics: `config-schema.md`): `true` → per task `feat({phase}): {task name}`; `false` → batch once per wave `feat({phase}): {wave desc} [tasks: ...]`.
- **Worktree** (`parallelization: true`): sequential creation (avoids `.git/config.lock` race), merge back after wave, squash to one commit/wave; restore STATE.md + ROADMAP.md from main after merge (prevents stale overwrite).
- **Failure recovery:** report failed plan → user picks retry / skip / abort; partial progress in STATE.md.

### UAT + Verification (STEP 8)

- **8a UAT:** user tests each AC; AI does not claim "done". AI creates `.planning/{phase}-UAT.md`, **shows expected behavior and asks if reality matches**, infers severity from the description. Fail → parallel debug agents → fix plans → Plan Checker → back to Step 7.
- **8b Goal-Backward Verification:** cross-reference `must_haves` with implementation artifacts → `.planning/{phase}-VERIFICATION.md`. Gaps → gap-closure plans → Step 7.
- **Mode B:** regression gate (run prior-phase tests) + schema-drift check (TS build + DB schema in sync).

### Brownfield bootstrap (STEP 0)

Map current stack, conventions, architecture patterns. If a migration is involved, inventory runtime state: stored data, live config, OS registration, secrets, build artifacts. Create state files from the mapping.

### Knowledge layer (STEP 0.5 + consumption hooks)

**STEP 0.5 — Knowledge Bootstrap (optional, once per project):** build `.claude/skills/<project>-*` reference skills (architecture-contract, change-control, debugging-playbook, config-and-flags, failure-archaeology, validation-and-qa, …) via the `mining-project-knowledge` skill (`/mine-knowledge`). Runs as its own campaign through this workflow.

Once the library exists, these **conditional hooks** fire (no-op without a library):

| Step | Hook |
|---|---|
| 1 Fast Lane | task touches an area `<project>-change-control` flags dangerous → not eligible |
| 4 Research | load `<project>-*` knowledge skills first; research only the gaps |
| 7 Execute | on BLOCKED, consult `<project>-debugging-playbook` before escalating model |
| 9 QA | use `<project>-validation-and-qa` as the evidence bar |
| 11 Ship | Knowledge Sync — write lessons back (failure-archaeology, debugging-playbook, config-and-flags, architecture-contract) |

Hooks are mirrored in `CLAUDE.md` (STEP 1/4/7/9/11) — keep both in sync.
