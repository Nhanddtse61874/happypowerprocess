# Mode Selection Criteria

This file is the authoritative reference for evaluating task complexity and selecting a runtime mode.
Claude MUST read this file during the Mode Selection Gate and score each criterion before suggesting a mode.

## When to Run

After brainstorm output is complete (or after Fast Lane assessment for Fast Lane tasks).
This gate is MANDATORY. It cannot be skipped except when Fast Lane marks brainstorm as skipped —
in that case, run this gate with the Fast Lane assessment as input.

## Scoring Criteria

Score each criterion from the brainstorm output. Add up Mode B signals.

### Criterion 1: Domain Count
- 1 domain (e.g., only .NET, or only React) → Mode A signal
- 2+ domains (e.g., .NET + React, React Native + IoT) → Mode B signal

### Criterion 2: Risk Level
- Low risk: isolated change, no production data, easy rollback → Mode A signal
- Medium risk: affects shared services or APIs → neutral
- High risk: affects auth, data migration, production infra, security → Mode B signal

### Criterion 3: QA / DevOps Gate Needed
- No formal QA gate needed, reviewer sign-off is enough → Mode A signal
- Formal QA review with severity classification required → Mode B signal
- CI/CD pipeline design or deployment strategy needed → Mode B signal

### Criterion 4: Cross-Team Coordination
- Single owner, no handoff needed → Mode A signal
- Multiple roles needed (architect + implementer + QA + DevOps) → Mode B signal

### Criterion 5: Output Format Required
- Informal, conversational outputs sufficient → Mode A signal
- Machine-readable JSON contracts required (for CI, automation, audit) → Mode B signal

## Threshold Rules

- 0–1 Mode B signals → suggest Mode A
- 2 Mode B signals → suggest Mode A with note that Mode B is viable
- 3+ Mode B signals → suggest Mode B

## Hard Exclusions (Always Mode B)

These cases require Mode B regardless of other scores:
- Task involves 3+ technical domains
- Task requires a formal release gate (CI/CD pipeline sign-off)
- Task has compliance, security audit, or data migration impact
- Task requires explicit architecture decision records (ADRs)
- Task involves multiple implementer agents working in parallel

## Reverse Hard Exclusion (Always Force Mode A) — v5.6.0+

Beyond the Hard Exclusions above (which force Mode B), v5.6.0 adds the reverse: certain conditions force Mode A regardless of 5-criteria score.

### Active harness detection

Mode B requires Claude Code's Task tool with `subagent_type` + `model` + `isolation` parameters. On other harnesses, Mode B partially fails (subagent dispatch differs, model names don't transfer).

Detection order:

1. Read `.planning/config.json` field `active_harness`. If explicit value (not `auto`), use that.
2. If `auto`, check environment variables:
   - `$env:CLAUDECODE` set (Windows) / `$CLAUDECODE` (POSIX) → `claude-code`
   - `$env:CURSOR_*` or `cursor` in process tree → `cursor`
   - `$env:CODEX_*` or `codex` in process tree → `codex`
   - `$env:OPENCODE_*` set → `opencode`
   - None match → fallback `claude-code` (preserves current behavior for ambiguous cases)
3. If detected harness != `claude-code`:
   - Force suggested mode = Mode A
   - Output reason in Mode Gate output: `"Mode B requires Claude Code Task tool. Detected harness: <name>. Forcing Mode A. Override with full understanding of partial-failure risk."`
4. User can override per existing user-decision-wins rule. On override to Mode B with non-Claude harness, AI MUST prepend warning to the override confirmation:
   ```
   ⚠ You are forcing Mode B on <harness>. Subagent dispatch and model
   selection may not work as designed. Proceed with manual review fallback ready.
   ```

See `docs/claude/harness-compatibility.md` for full compatibility matrix.

## Mode Selection Gate Output Format

Present this to the user after scoring:

```
Mode Selection Evaluation
─────────────────────────
Domain count:             [1 / 2 / 3+]        → [Mode A / Mode B] signal
Risk level:               [low / medium / high] → [Mode A / neutral / Mode B] signal
QA/DevOps gate needed:    [yes / no]           → [Mode B / Mode A] signal
Cross-team coordination:  [yes / no]           → [Mode B / Mode A] signal
Output format required:   [formal / informal]  → [Mode B / Mode A] signal

Mode B signals: [N] / 5
Suggested mode: [A / B]
Reason: [one sentence citing the dominant signals]

Do you approve Mode [A/B], or would you like to override?
```

## Borderline Examples

**Example 1 — Clearly Mode A:**
Task: Fix a null reference bug in a .NET service.
- Domain count: 1 (.NET) → A
- Risk: low (isolated bug fix) → A
- QA gate: no → A
- Cross-team: no → A
- Output format: informal → A
→ 0 Mode B signals → **suggest Mode A**

**Example 2 — Borderline (Mode A viable, Mode B reasonable):**
Task: Add a new API endpoint in .NET + update the React dashboard to consume it.
- Domain count: 2 (.NET + React) → B
- Risk: medium (shared API) → neutral
- QA gate: no formal gate needed → A
- Cross-team: no → A
- Output format: informal → A
→ 1 Mode B signal → **suggest Mode A**, note Mode B is viable if stricter review needed

**Example 3 — Clearly Mode B:**
Task: Migrate authentication system from JWT to OAuth2, update .NET backend + React Native app.
- Domain count: 2 (.NET + React Native) → B
- Risk: high (auth, security) → B
- QA gate: yes (security review required) → B
- Cross-team: yes (architect + implementer + QA) → B
- Output format: formal (audit trail needed) → B
→ 5 Mode B signals + hard exclusion (security audit) → **suggest Mode B**

## Override Rule

User decision always wins. Once the user approves or overrides:
- Their chosen mode is the source of truth for ALL downstream phases.
- Claude does not re-evaluate or adjust after user approval.
- Spec, Plan, Implementation, QA, and Finish all follow the user-approved mode without exception.
