# Runtime Modes & Mode Selection

Authoritative reference for the plugin's two runtime modes **and** the Mode Selection Gate (STEP 3). Claude MUST read this file during the Mode Selection Gate and score each criterion before suggesting a mode.

*(Consolidates the former `mode-selection-criteria.md`, `runtime-modes.md`, and `harness-compatibility.md`.)*

---

## Mode A — Solo (Superpowers)

A lighter workflow without agent-team orchestration. Use for single-domain, lower-complexity tasks with no formal QA gate.

Execution backbone:
- `skills/using-superpowers/SKILL.md`
- `skills/brainstorming/SKILL.md`
- `skills/writing-plans/SKILL.md`
- `skills/subagent-driven-development/SKILL.md` or `skills/executing-plans/SKILL.md`
- `skills/test-driven-development/SKILL.md`
- `skills/requesting-code-review/SKILL.md`
- `skills/finishing-a-development-branch/SKILL.md`

Characteristics: no dependency on `agents/*.md`; faster setup for small/medium tasks; strong process discipline from Superpowers skills; mandatory stack skill still applies (see `stack-skill-rule-map.md`).

## Mode B — AI Team Spine

Role-based execution with explicit ownership across phases. Use for multi-domain, high-risk, or formal-QA/DevOps-gate work.

Execution backbone: all Mode A skills **plus** the team orchestration layer — `agents/team-orchestrator.md`, `agents/master-dispatcher.md`, and specialist/implementer agents selected by the dispatcher.

Characteristics: clear role ownership and phase gates; structured outputs with `template_id` contracts (see `agent-output-templates.md`); better control for larger cross-domain work.

**Team topology, phase model, and dispatch routing:** see `ai-team.md`.

---

## Mode Selection Gate (STEP 3)

### When to run

After brainstorm output is complete (or after Fast Lane assessment for Fast Lane tasks). This gate is **MANDATORY**. It cannot be skipped except when Fast Lane marks brainstorm as skipped — in that case, run this gate with the Fast Lane assessment as input.

Run the gate after **every** brainstorm. Do not select a mode manually without running the gate first.

### Scoring criteria

Score each criterion from the brainstorm output. Add up Mode B signals.

| # | Criterion | Mode A signal | Mode B signal |
|---|---|---|---|
| 1 | Domain count | 1 domain (e.g. only .NET) | 2+ domains (e.g. .NET + React) |
| 2 | Risk level | low: isolated, no prod data, easy rollback | high: auth, data migration, prod infra, security *(medium = neutral)* |
| 3 | QA / DevOps gate | reviewer sign-off is enough | formal severity-classified QA, or CI/CD pipeline design |
| 4 | Cross-team coordination | single owner, no handoff | multiple roles (architect + implementer + QA + DevOps) |
| 5 | Output format | informal, conversational | machine-readable JSON contracts (CI, automation, audit) |

### Threshold rules

- 0–1 Mode B signals → suggest **Mode A**
- 2 Mode B signals → suggest **Mode A**, note that Mode B is viable if stricter review needed
- 3+ Mode B signals → suggest **Mode B**

### Hard Exclusions (always Mode B)

Regardless of score:
- Task involves 3+ technical domains
- Task requires a formal release gate (CI/CD pipeline sign-off)
- Task has compliance, security audit, or data migration impact
- Task requires explicit architecture decision records (ADRs)
- Task involves multiple implementer agents working in parallel

### Gate output format

Present this to the user after scoring:

```
Mode Selection Evaluation
─────────────────────────
Domain count:             [1 / 2 / 3+]          → [Mode A / Mode B] signal
Risk level:               [low / medium / high] → [Mode A / neutral / Mode B] signal
QA/DevOps gate needed:    [yes / no]            → [Mode B / Mode A] signal
Cross-team coordination:  [yes / no]            → [Mode B / Mode A] signal
Output format required:   [formal / informal]   → [Mode B / Mode A] signal

Mode B signals: [N] / 5
Suggested mode: [A / B]
Reason: [one sentence citing the dominant signals]

Do you approve Mode [A/B], or would you like to override?
```

### Borderline examples

**Example 1 — Clearly Mode A:** Fix a null reference bug in a .NET service.
Domain 1 (.NET)→A · Risk low→A · QA no→A · Cross-team no→A · Output informal→A → **0 B signals → Mode A**

**Example 2 — Borderline:** Add a new .NET API endpoint + update the React dashboard to consume it.
Domain 2→B · Risk medium→neutral · QA no→A · Cross-team no→A · Output informal→A → **1 B signal → Mode A** (Mode B viable if stricter review needed)

**Example 3 — Clearly Mode B:** Migrate auth from JWT to OAuth2 across .NET backend + React Native app.
Domain 2→B · Risk high (security)→B · QA yes→B · Cross-team yes→B · Output formal→B → **5 B signals + hard exclusion (security audit) → Mode B**

### Override rule

User decision always wins. Once the user approves or overrides:
- Their chosen mode is the source of truth for ALL downstream phases.
- Claude does not re-evaluate or adjust after user approval.
- Spec, Plan, Implementation, QA, and Finish all follow the user-approved mode without exception.

---

## Harness Compatibility — Mode B is Claude Code-only (v5.6.0+)

### Summary matrix

| Workflow | Claude Code | Cursor | Codex | OpenCode |
|---|---|---|---|---|
| **Mode A (Solo)** | ✅ Full | ✅ Full | ✅ Full | ✅ Full |
| **Mode B (Team Spine)** | ✅ Full | ❌ Not supported | ❌ Not supported | ❌ Not supported |
| `/init-project`, `/update-config`, `/add-tech-stack`, `/sync-stack-skill` | ✅ slash command | ✅ slash command | ⚠ invoke skill by name | ✅ |
| Hooks | ✅ `.claude/settings.json` | ⚠ `hooks-cursor.json` | ❌ no native hooks | ⚠ runtime-specific |
| Worktree parallelization | ✅ | ⚠ harness-dependent | ⚠ harness-dependent | ⚠ harness-dependent |

### Why Mode B is Claude-only

Mode B requires three Claude-specific capabilities:
1. **Task tool with `subagent_type` + `model` + `isolation` parameters** — other harnesses' agent APIs differ; cannot guarantee fresh isolated context per subagent dispatch.
2. **Per-task model override using Claude model names** (`haiku` / `sonnet` / `opus`) — Codex uses GPT models, Cursor depends on user API config; literal Claude names don't transfer.
3. **Worktree-isolated parallel subagent execution** — parallelization mechanism varies; may not isolate properly.

Without all three, Mode B partial-fails: phase agents may load but dispatch behavior diverges, model selection silently degrades, parallel execution serializes or conflicts.

### Reverse Hard Exclusion — non-Claude harness forces Mode A

Beyond the Hard Exclusions above (which force Mode B), certain conditions force **Mode A** regardless of the 5-criteria score.

Detection order:
1. Read `.planning/config.json` field `active_harness`. If explicit (not `auto`), use that.
2. If `auto`, check environment variables:
   - `$env:CLAUDECODE` (Windows) / `$CLAUDECODE` (POSIX) → `claude-code`
   - `$env:CURSOR_*` or `cursor` in process tree → `cursor`
   - `$env:CODEX_*` or `codex` in process tree → `codex`
   - `$env:OPENCODE_*` → `opencode`
   - None match → fallback `claude-code` (conservative default keeps Mode B available for ambiguous cases)
3. If detected harness != `claude-code`: force suggested mode = Mode A, and output in the Mode Gate:
   `"Mode B requires Claude Code Task tool. Detected harness: <name>. Forcing Mode A. Override with full understanding of partial-failure risk."`
4. User can override per the user-decision-wins rule. On override to Mode B with a non-Claude harness, AI MUST prepend:
   ```
   ⚠ You are forcing Mode B on <harness>. Subagent dispatch and model
   selection may not work as designed. Proceed with manual review fallback ready.
   ```

### Workaround for non-Claude harnesses

For complex tasks that would normally trigger Mode B on Claude:
- **Run Mode A solo.** All 11 workflow steps still apply.
- **Add manual review checkpoints** after each major step instead of via `qa-code-reviewer` subagent.
- **Do QA Gate manually** — run `requesting-code-review` skill directly instead of dispatched.
- **Sequential execution** — no parallelization; one task at a time.
- **Document decisions inline** — Mode B's formal artifact templates can be filled manually if an audit trail is needed.

Outputs are slightly less formal but functionally equivalent for most projects.

> Cross-platform Mode B emulation (model-tier abstraction, harness adapter layer) is deferred indefinitely — multi-month effort for a solo maintainer. May reconsider at v7.0+.

---

## Cross-cutting rules (both modes)

These apply in Mode A **and** Mode B. Each has one authoritative home — this section is only a pointer index:

| Rule | Source of truth |
|---|---|
| Stack/domain skill mandatory for every implementation task | `stack-skill-rule-map.md` |
| Per-task model assignment (`<model>` = haiku/sonnet/opus), `model_defaults`, `model_profile` shift | `config-schema.md` |
| Commit configuration (`commit_docs`, `commit_atomic`) | `config-schema.md` |
| Fast Lane (hotfix/small task) eligibility + skips | `current-process-workflow.md` STEP 1 |
| Output template contracts (Mode B) | `agent-output-templates.md` |
