# Unified Workflow: GSD + Superpowers (v3)

Combines GSD state management, research phase, wave-based execution, UAT gate, and milestone rhythm with Superpowers Mode A/B, Fast Lane, QA gate, DevOps phase, and escalation protocol.

**Principle:** AI assists through each step — never auto-advances without human approval.

Reference files:
- State files: `docs/claude/state-files-guide.md`
- Templates: `docs/claude/templates/`
- Research phase: `docs/claude/research-phase-guide.md`
- Mode selection: `docs/claude/mode-selection-criteria.md`
- Agent templates: `docs/claude/agent-output-templates.md`
- Stack skill map: `docs/claude/stack-skill-rule-map.md`

---

## STEP 0 — Project Bootstrap

**Purpose:** Set up or read project state for full context before doing anything.

**Artifact persistence principle:** Every artifact must be written to disk immediately — if context is lost, artifacts survive and work can resume.

### New project

1. Detect greenfield vs brownfield (`.git` exists? codebase exists?)
2. Set up git if needed
3. Collect config upfront — save to `.planning/config.json`:
   - **mode**: `interactive` (pause for confirmation each step) or `yolo` (auto-approve)
   - **granularity**: `coarse` (3-5 phases) / `standard` (5-8) / `fine` (8-12+)
   - **parallelization**: `true` / `false`
   - **commit_docs**: object controlling git commit behavior:
     - `state_files`: commit STATE.md, PROJECT.md, REQUIREMENTS.md, ROADMAP.md (default `true`)
     - `planning_artifacts`: commit `.planning/` files — research, plans, UAT, summaries (default `true`). When `false`: `.planning/` still created on disk but auto-added to `.gitignore`
   - **commit_atomic**: `true` (commit after each task — default) or `false` (batch — commit once per wave/phase). Confirmed again before execution starts
   - **model_profile**: `balanced` / `quality` / `budget` — default strategy for model selection
   - **model_defaults**: maps complexity tier to specific model name:
     - `mechanical`: for simple tasks (1-2 files, clear spec) — default `haiku`
     - `standard`: for integration tasks (multi-file, pattern matching) — default `sonnet`
     - `complex`: for complex tasks (architecture, design judgment) — default `opus`
     - `model_profile` shifts tiers: `budget` → all down one, `quality` → all up one. Clamped at boundaries (haiku cannot go lower, opus cannot go higher)
   - **workflow.research**: enable research phase
   - **workflow.plan_check**: enable plan checker
4. **Deep questioning** (interactive mode): freeform questions about project vision — no checklist, follow natural conversational threads. Probe:
   - What excites them about this idea
   - Specific problem being solved
   - Clarify vague terms with concrete examples
   - Existing decisions already made
   - Decision gate: when context is sufficient → create PROJECT.md
5. Create state files from templates (`docs/claude/templates/`):
   - `PROJECT.md`, `REQUIREMENTS.md` with REQ-IDs, `ROADMAP.md`, `STATE.md`
   - ROADMAP.md phase count must match `granularity` config: `coarse` = 3-5 phases, `standard` = 5-8, `fine` = 8-12+
   - Create `.planning/` and `.planning/research/` directories
6. Commit artifacts per `commit_docs` config:
   - `state_files: true` (default): commit PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md
   - `planning_artifacts: true` (default): commit `.planning/` files
   - `planning_artifacts: false`: add `.planning/` to `.gitignore`, commit state files only
   - Commit message: `git commit -m "init: project state files"`

### Brownfield project

1. Map codebase: analyze current stack, conventions, architecture patterns
2. Runtime state inventory if migration involved: stored data, live config, OS registration, secrets, build artifacts
3. Create state files from mapping results

### Resuming session

**FIRST: Run `skills/validate-state/SKILL.md`** — mandatory safety net before reading state files. Reports any drift, schema errors, or freshness gaps. On `FAIL`, must fix before continuing. On `WARN`, present report and let user choose `accept_drift / fix_now / cancel`. On `CLEAN`, continue immediately.

- Read `STATE.md` → know current step, next action, blockers
- Read `.planning/config.json` → display current config to user:

```
Current config:
─────────────────────────────────
mode:               interactive
model_profile:      balanced
model_defaults:     haiku / sonnet / opus
commit_atomic:      true
commit_docs:        state_files=true, planning_artifacts=true
parallelization:    true
workflow:           research=true, plan_check=true

Want to change anything? [keep / edit]
```

- If user picks `edit` → ask which fields to change, update `.planning/config.json`
- If user picks `keep` → continue from step in STATE.md, no restart

**Human touchpoint:** User confirms project context and config before continuing.

**Output:** `.planning/config.json`, `PROJECT.md`, `REQUIREMENTS.md` (with REQ-IDs), `ROADMAP.md`, `STATE.md`

---

## STEP 1 — Fast Lane Check

**Purpose:** Evaluate whether the task qualifies for Fast Lane.

**Fast Lane criteria (all must be true):**
- Fix scope is clear and unambiguous
- Small change (1-2 files)
- No architecture/API contract change
- No security or data migration impact
- Low regression risk

**If eligible:** Skip Steps 2-4, go directly to Step 5 (Spec) with Fast Lane output as input
**If not eligible:** Continue to Step 2

**Template output:** `fast-lane-assessment-v1`

**Human touchpoint:** User confirms Fast Lane or full workflow.

---

## STEP 2 — Brainstorm

**Purpose:** Understand requirements, scope, and constraints. Propose 2-3 approaches with trade-offs.

**Skill:** `skills/brainstorming/SKILL.md`

**Output:**
- Problem statement, scope, constraints, success criteria
- 2-3 approaches with trade-offs
- Approved design direction
- Updated REQUIREMENTS.md — requirements must have REQ-IDs, be testable, user-centric, atomic

**REQ-ID rule:** Every v1 requirement must map to exactly one phase in ROADMAP.md. 100% coverage required. Phase count must stay within `granularity` range from config.

**Human touchpoint:** User approves design direction. Do not continue without approval.

---

## STEP 3 — Mode Selection Gate

**Purpose:** Score 5 criteria from brainstorm output, suggest Mode A or B.

**Reference:** `docs/claude/mode-selection-criteria.md`

**Scoring criteria:**
1. Domain count (1 vs 2+)
2. Risk level (low/medium/high)
3. QA/DevOps gate needed (yes/no)
4. Cross-team coordination (yes/no)
5. Output format (formal/informal)

**Threshold:** 0-1 B signals → Mode A | 2 B signals → Mode A (Mode B viable) | 3+ → Mode B

**Mode A:** Solo, lightweight, single domain
**Mode B:** AI team spine, multi-domain, formal QA/DevOps gate, role-based ownership

**Human touchpoint:** User approves mode. Mode is locked after approval — this is the source of truth.

---

## STEP 4 — Research

**Purpose:** Parallel research agents gather evidence before writing spec. Plans must be evidence-based, not assumption-based.

**Reference:** `docs/claude/research-phase-guide.md`

**Skip if:** Fast Lane eligible or `workflow.research: false` in config.

### Critical startup (each agent)

Before researching:
1. Load CONTEXT.md — locked decisions constrain scope
2. Load `.planning/config.json` — validation settings
3. Read CLAUDE.md — project directives override recommendations

### Mode A: 2 parallel agents

| Agent | Output |
|---|---|
| Stack Researcher | Stack findings with [VERIFIED/CITED/ASSUMED] tags |
| Pitfall Researcher | Anti-patterns, gotchas, common mistakes |

**Output:** `.planning/{phase}-RESEARCH.md`

### Mode B: 4 parallel agents + Synthesizer

| Agent | Output file |
|---|---|
| Stack Researcher | `.planning/research/STACK.md` |
| Feature Researcher | `.planning/research/FEATURES.md` |
| Architecture Researcher | `.planning/research/ARCHITECTURE.md` |
| Pitfall Researcher | `.planning/research/PITFALLS.md` |
| **Research Synthesizer** (runs after) | `.planning/research/SUMMARY.md` |

**Claim provenance (required):** Every claim must be tagged `[VERIFIED]`, `[CITED]`, or `[ASSUMED]`. Never present assumed knowledge as fact.

**Human touchpoint:** User reviews research output, confirms sufficiency before continuing to Step 5.
If gaps exist: add targeted research agent — do not restart the entire phase.

---

## STEP 5 — Spec Writing

**Purpose:** Write technical spec from brainstorm output + research findings.

**Mode A:** `skills/brainstorming/SKILL.md` — solo spec writing
**Mode B:**
- `agents/phase-discovery-lead.md`: formalize requirements spec from brainstorm output + RESEARCH.md
- `agents/phase-architecture-lead.md`: formalize technical spec (contracts, interfaces, trade-offs) from brainstorm + research
- Input: brainstorm output + research findings (do not re-brainstorm)
- Output templates: `phase-lead-report-v1`

**Output:** `docs/superpowers/specs/YYYY-MM-DD-{topic}-design.md`

**Human touchpoint:** User approves spec. If changes needed → revise spec, do not re-brainstorm.

---

## STEP 6 — Planning

**Purpose:** Write implementation plan using goal-backward methodology, XML tasks grouped by waves.

**Mode A:** `skills/writing-plans/SKILL.md`
**Mode B:** `agents/phase-implementation-lead.md`

### Goal-Backward Methodology (required)

Before writing tasks, derive from phase goal:

1. **Observable Truths** (3-7, user perspective): "User can see messages", "User can send message"
2. **Required Artifacts** (specific files): `src/components/Chat.tsx`, `src/api/chat/route.ts`
3. **Required Wiring** (connections): Chat component fetches from /api/chat on mount
4. **Key Links** (critical failure points): Where breakage cascades

→ Output into `must_haves` frontmatter of PLAN.md

### Task Structure

See `docs/claude/templates/plan-task.xml` for full template.

Each task must have:
- `<model>`: Specific model name (`haiku` / `sonnet` / `opus`) — planner assigns based on complexity signals from `model_defaults` in config.json. User can override per task before approving the plan
- `<read_first>`: Files executor must read before starting
- `<action>`: Specific implementation — include WHAT to AVOID and why, corresponding REQ-ID
- `<verify><automated>`: Command completing in <60 seconds (Nyquist Rule)
- `<done>`: Measurable, grep-verifiable criteria

**Model assignment signals for planner:**
- 1-2 files, clear spec, mechanical work → `haiku` (or `model_defaults.mechanical`)
- Multi-file, integration, pattern matching → `sonnet` (or `model_defaults.standard`)
- Architecture, design judgment, broad codebase → `opus` (or `model_defaults.complex`)

### Context Budget Rule

- Target ~50% token usage per plan
- Maximum 2-3 tasks per plan
- If scope exceeds budget → split into smaller plans, **never compress**

### Wave Assignment Algorithm

```
if task.depends_on is empty → Wave 1
else → Wave = max(wave of dependencies) + 1
if task.files_modified overlap with earlier task → force later wave
```

Same-wave plans must have **zero file overlap** — required.

### Plan Checker (if `workflow.plan_check: true`)

After plan is written, Plan Checker validates 11 dimensions before approval:

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

**Scope Reduction Detection:** If plan references user decisions but delivers "v1 stubs" → **always blocker**.

**Revision loop:** Max 3 iterations. If issue count does not decrease → escalate to user.

**Output:** `.planning/{phase}-{N}-PLAN.md`

**Human touchpoint:** User approves plan + wave structure before execution.

---

## STEP 7 — Execution (Wave by Wave)

**Purpose:** Execute each wave with fresh independent context per task.

**Skill:** `skills/subagent-driven-development/SKILL.md` or `skills/executing-plans/SKILL.md`

### Intra-wave Safety Check

Before executing a wave:
- Check file overlap between plans in the same wave
- If 2 plans modify the same file → force sequential, not parallel

### Pre-Task Confirmation (interactive mode)

Before dispatching each task, controller notifies the user:

```
Task: {task name}
Model: {haiku/sonnet/opus} (from plan)
Files: {files_modified}

Proceed with this model, or switch? [proceed / haiku / sonnet / opus]
```

- `proceed` → dispatch with the model from the plan
- User picks a different model → override for this task only (does not change the plan)
- When `mode: yolo` → skip confirmation, use model from plan

### Fresh Context per Task

- Each subagent receives **only the required files** from `<read_first>` — not the entire codebase
- Fresh 200k-token context per task — eliminates context rot
- **Model dispatch**: Controller reads `<model>` from task XML (or user override from pre-task confirmation) and dispatches subagent with that model. If subagent is BLOCKED, controller can escalate to a higher tier (e.g., haiku → sonnet → opus)

### Worktree Isolation

When `parallelization: true`:
- Each executor runs in an isolated git worktree
- Worktree creation must be **sequential** (avoids `.git/config.lock` race condition)
- After wave: merge worktree branches back to main
- **File protection**: Restore STATE.md, ROADMAP.md from main branch after merge — prevents stale version overwrite

### Commit Strategy

Before executing the first wave, controller confirms with user:

```
Commit strategy: {atomic/batch} (from config — commit_atomic: {true/false})
- atomic (true): commit after each task — bisectable, revertable
- batch (false): commit once after wave completes

Keep this strategy, or change? [keep / atomic / batch]
```

**When `commit_atomic: true`** (default):
Task complete → commit **immediately**:
```
feat({phase}): {task name}
```
Each task is an independent commit — bisectable, revertable.

**When `commit_atomic: false`** (batch):
- Task complete → stage changes but **do not commit**
- Commit once after the entire wave completes:
  ```
  feat({phase}): {wave description} [tasks: {task1}, {task2}, ...]
  ```
- Trade-off: loses bisectability but reduces commit noise
- **Worktree interaction:** When `parallelization: true`, worktrees still commit internally (git requires commits to merge). On merge back to main, squash into a single commit per wave. User sees one commit on main per wave — achieves batch intent while preserving parallel execution

**When `mode: yolo`** → skip confirmation, use strategy from config.

### Stack Skill (required for both modes)

Per `docs/claude/stack-skill-rule-map.md`:
- `.NET/C#` → `skills/implementer-dotnet-csharp/SKILL.md`
- `React Native` → `skills/implementer-react-native-typescript/SKILL.md`
- `Angular` → `skills/implementer-angular-typescript/SKILL.md`
- `React` → `skills/implementer-react-typescript/SKILL.md`
- `IoT/MQTT/BLE` → `skills/implementer-iot-edge/SKILL.md`

### Mode B Stack Agents

Dispatcher selects implementer agent by stack:
- `agents/implementer-react-native-typescript.md`
- `agents/implementer-dotnet-csharp.md`
- `agents/implementer-angular-typescript.md`
- `agents/implementer-react-typescript.md`
- `agents/implementer-iot-edge.md`
- Output: `implementer-delivery-v1`

### Failure Recovery

If executor fails (missing SUMMARY.md, test failure):
- Report which plan failed
- User decides: retry / skip / abort
- Partial progress recorded in STATE.md for resume

**Human touchpoint:** User approves results of each wave before the next wave runs.

---

## STEP 8 — UAT + Verification

**Two separate parts:**

### 8a — UAT (User Acceptance Testing)

**Purpose:** User tests the feature against acceptance criteria. AI does not claim "done".

- AI creates `.planning/{phase}-UAT.md` from template (`docs/claude/templates/phase-UAT.md`)
- User runs the feature, tests each AC, records results in plain text
- **Show expected, ask if reality matches** — AI describes expected behavior, user confirms or describes differences
- AI infers severity from user's description — no questionnaire

**Pass:** Continue to 8b
**Fail:** AI spawns parallel debug agents → diagnose root causes → planner creates fix plans → Plan Checker verifies (max 3 revision cycles) → back to Step 7

**Template output:** `uat-gate-v1`

### 8b — Goal-Backward Verification

**Purpose:** AI cross-references implementation artifacts with phase must_haves.

- Extract success criteria from phase goal in ROADMAP.md
- Cross-reference with SUMMARY.md files from execution
- Create `.planning/{phase}-VERIFICATION.md` (see `docs/claude/templates/phase-VERIFICATION.md`)

**Gaps found:** Offer gap closure → create gap-closure plans → back to Step 7

**Regression gate (Mode B):** Run prior phase test suites to catch cross-phase regressions.

**Schema drift detection (if relevant):** Verify TypeScript build passes and DB schema remains in sync.

**Human touchpoint:** UAT is entirely human-driven. Verification is AI-assisted but user makes the final decision.

---

## STEP 9 — QA Gate

**Purpose:** Formal QA review with severity classification.

**Mode A:** `skills/requesting-code-review/SKILL.md`
**Mode B:** `agents/phase-qa-gate.md` + `agents/qa-code-reviewer.md`

**Findings severity:** Critical | Important | Suggestion

**Block:** Fix → back to Step 7 with targeted fix plan
**Approve / Approve with conditions:** Continue to Step 10

**Template output:** `qa-review-v1`

**Human touchpoint:** User reviews QA findings, decides whether to fix or approve-with-conditions.

---

## STEP 10 — Release / DevOps

**Purpose:** CI/CD plan, deployment strategy, rollback, observability.

**Mode A:** `skills/finishing-a-development-branch/SKILL.md`
**Mode B:** `agents/phase-release-devops-lead.md` + `agents/devops-cicd-assistant.md`

**Template output (Mode B):** `devops-release-v1`

**Human touchpoint:** User approves release readiness before shipping.

**Skip if:** Mode A and task does not need a formal release gate.

---

## STEP 11 — Ship & Milestone Close

**Purpose:** Merge/PR + update state files + open new milestone if needed.

**Process:**
1. Create PR or merge per strategy from Step 10
2. Write `.planning/{phase}-SUMMARY.md` from template — records: what was done, decisions made, commits, lessons, residual risks
3. Update ROADMAP.md: mark milestone as done
4. Update STATE.md: current position = start of next milestone (or project complete)
5. Escalation: if unresolved risk/conflict exists → present 2-3 options for user to decide

**Template output:** `phase-summary-v1`
**Mode B:** `agents/team-orchestrator.md` compiles final handoff (`orchestrator-status-v1`)

**Human touchpoint:** User decides to ship, reviews summary, decides next milestone.

---

## Config Update (Anytime)

User can request "update config" at any point in the workflow. AI will:

1. Read current `.planning/config.json`
2. Display all settings in a table
3. Ask which fields to change
4. Update `.planning/config.json` immediately
5. If changes affect already-completed steps (e.g., changing `model_defaults` after plan is written) → warn user and suggest re-plan if needed

**Trigger phrases:** "update config", "change config", "change model", "change commit strategy"

**Mid-workflow impact of config changes:**

| Field | Impact if changed mid-workflow |
|---|---|
| `mode` | Affects human touchpoints only, not already-completed steps |
| `model_profile` / `model_defaults` | Unexecuted tasks use new model. Executed tasks unchanged |
| `commit_atomic` | Applies from next task. Existing commits unchanged |
| `commit_docs` | Applies from next commit. If `planning_artifacts` changed → update `.gitignore` |
| `parallelization` | Applies from next wave |
| `workflow.*` | Applies from next step |
| `granularity` | Affects unplanned phases only. Already-planned phases unchanged |

---

## Fast Lane Path

```
STEP 0 (Bootstrap) → STEP 1 (Fast Lane: eligible) → STEP 5 (Spec) → STEP 6 (Plan)
→ STEP 7 (Execute) → STEP 8 (UAT + Verification) → STEP 9 (QA) → STEP 11 (Ship)
```

Skip: Steps 2, 3, 4, 10 (unless Mode B and formal release gate needed)

---

## Full Path Summary

```
STEP 0:  Bootstrap          → config.json, state files, deep questioning  [USER CONFIRMS]
STEP 1:  Fast Lane?         → eligible: skip to STEP 5                    [USER CONFIRMS]
STEP 2:  Brainstorm         → approved direction, REQ-IDs updated          [USER APPROVES]
STEP 3:  Mode Gate          → Mode A or B                                  [USER APPROVES]
STEP 4:  Research           → parallel agents + synthesizer, claim tags    [USER REVIEWS]
STEP 5:  Spec               → solo (A) / team agents (B)                  [USER APPROVES]
STEP 6:  Plan               → goal-backward, XML tasks, wave groups        [USER APPROVES]
          └─ Plan Checker   → 11-dimension validation, max 3 revision loops
STEP 7:  Execute            → wave by wave, fresh context, model confirm   [USER APPROVES each wave]
          └─ Pre-task model confirm, commit strategy confirm, worktree isolation, failure recovery
STEP 8:  UAT + Verify       → user tests + goal-backward verification      [USER DRIVES]
          └─ Gap closure loop, regression gate, schema drift detection
STEP 9:  QA Gate            → severity findings + fix loop                 [USER APPROVES]
STEP 10: Release/DevOps     → CI/CD plan (Mode B)                         [USER APPROVES]
STEP 11: Ship               → PR/merge + SUMMARY + ROADMAP + STATE updated [USER DECIDES]
```

---

## Output Standards

- Mode A: Superpowers skill outputs
- Mode B: `template_id` from `docs/claude/agent-output-templates.md` (required)
- Both: commit behavior controlled by `commit_atomic` + `commit_docs` config
- Both: artifact persistence — write to disk immediately, never hold in memory only
- Both: state files updated after each step completes
- Both: `<model>` required in every task XML — planner assigns, user can override before approval
- REQ-ID traceability: every v1 requirement must map to at least one plan task