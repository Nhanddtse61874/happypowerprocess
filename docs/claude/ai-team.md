# AI Team (Mode B) — Topology & Dispatch Routing

Reference for **Mode B** only. It defines the reusable AI engineering team, the phase-based ownership model, and the master-dispatcher routing brain. Mode A does not use any of this.

*(Consolidates the former `full-ai-team-setup.md` and `master-dispatcher-prompt.md`.)*

For mode selection and when Mode B applies, see `modes.md`.

---

## 1. Goal

Let Claude operate like a real multi-role software team across projects: clean/consistent code, stack-specific best practice, early self-review, architecture + CI/CD design, stable behavior across sessions.

## 2. Team Topology

One orchestrator plus specialist agents:
- Team Orchestrator (`agents/team-orchestrator.md`)
- Master Dispatcher (`agents/master-dispatcher.md`)
- Senior React Native Engineer
- Backend Architect (.NET / C#)
- Frontend Engineer (Angular)
- Frontend Engineer (React)
- QA / Code Reviewer
- DevOps / CI-CD Assistant
- System Designer (MQTT / BLE / IoT)

## 3. Phase-Based Ownership Model

| Phase | Owner | Output |
|---|---|---|
| Discovery & Scope | `phase-discovery-lead` | clarified scope, constraints, acceptance criteria, risk list |
| Architecture & Design | `phase-architecture-lead` | architecture decisions, interfaces, trade-offs, rollout strategy |
| Implementation | `phase-implementation-lead` | incremental implementation with tests and change notes |
| QA & Review Gate | `phase-qa-gate` | severity-based findings, regression check, fix list |
| Release & Operations | `phase-release-devops-lead` | CI/CD plan, release checklist, rollback + observability plan |

Stack implementer agents (coding tasks): `implementer-react-native-typescript`, `implementer-dotnet-csharp`, `implementer-angular-typescript`, `implementer-react-typescript`, `implementer-iot-edge`. Each implementer must report: what changed, why this design, tests added/updated, risks + follow-ups. **Mandatory stack skills per stack: see `stack-skill-rule-map.md`.**

## 4. Operating Rules

- Always start with constraints and acceptance criteria.
- Prefer small, verifiable increments.
- Enforce tests before declaring completion.
- Separate architecture decisions from coding details.
- Keep review findings prioritized: Critical, Important, Suggestion.
- Never hide uncertainty; call out assumptions explicitly.

## 5. Standard Handoff Contract

Every agent handoff must include: Context (problem, scope, constraints) · Input artifacts (files, APIs, related decisions) · Expected output (exact deliverables) · Done criteria (objective checks) · Risks (top known risks + mitigations).

---

## 6. Master Dispatcher — Routing Brain

Route incoming work to the correct agent by task type, then enforce a standard output contract so results are machine-readable and review-friendly.

### Input the dispatcher receives
User objective · constraints (time, quality, stack, compliance) · current phase (if known) · existing artifacts (spec, plan, code, tests, pipeline) · brainstorm output artifact (present for all non-Fast-Lane tasks).

### Dispatch algorithm
1. Normalize request into one dominant `task_type` (+ optional secondary). If brainstorm output is present and task_type is spec or plan → use it as primary input; do not brainstorm again.
2. Detect current lifecycle phase: discovery, architecture, implementation, qa, release.
3. Run Fast Lane eligibility analysis (see `current-process-workflow.md` STEP 1).
4. Select owner agent by routing table.
5. If coding is required, choose stack-specific implementer.
6. Attach acceptance criteria and done checks.
7. Require output in a template from `agent-output-templates.md`.
8. If risk is high or unclear, escalate with options before execution.

### Routing table (task type → primary agent)

| task_type | Owner |
|---|---|
| `requirements-clarification`, `scope-definition`, `spec-discovery` | `phase-discovery-lead` |
| `architecture-design`, `spec-architecture` | `phase-architecture-lead` |
| `system-design-iot` | `system-designer-mqtt-ble-iot` |
| `implementation-planning`, `plan-writing` | `phase-implementation-lead` |
| `research-phase-mode-a` | main session (2 parallel subagents: stack + pitfall) |
| `research-phase-mode-b` | `team-orchestrator` (4 parallel subagents + synthesizer) |
| `uat-gate` | human (AI creates `.planning/{phase}-UAT.md`, user fills results) |
| `phase-summary` | main session or `team-orchestrator` (compile after Ship) |
| `implement-react-native` | `implementer-react-native-typescript` |
| `implement-dotnet-backend` | `implementer-dotnet-csharp` |
| `implement-angular` | `implementer-angular-typescript` |
| `implement-react` | `implementer-react-typescript` |
| `implement-iot-edge` | `implementer-iot-edge` |
| `code-review` | `qa-code-reviewer` |
| `qa-gate` | `phase-qa-gate` |
| `cicd-design` | `devops-cicd-assistant` |
| `release-readiness` | `phase-release-devops-lead` |
| `cross-phase-coordination` | `team-orchestrator` |

### Stack detector (keyword → implementer)
- React Native / mobile / app store / device API → `implementer-react-native-typescript`
- ASP.NET / .NET / C# / Web API / EF Core → `implementer-dotnet-csharp`
- Angular / modules / services / rxjs-heavy UI → `implementer-angular-typescript`
- React / hooks / Next-style SPA → `implementer-react-typescript`
- MQTT / BLE / edge gateway / device firmware → `implementer-iot-edge`

### Escalation
Escalate with 2–3 options when: requirements conflict with constraints · security/reliability trade-off affects production risk · timeline forces scope cuts · task needs an architectural change not yet approved.

### Execution task fields (implementation/research)
`wave_number` · `model` (haiku/sonnet/opus from task XML — dispatcher dispatches subagent with this model) · `fresh_context: true` · `atomic_commit` per `commit_atomic` config.

### Required handoff block
`task_type` · `selected_agent` · `objective` · `constraints` · `inputs` · `done_criteria` · `output_template` · `fast_lane_assessment`.

### Dispatcher output (JSON)

```json
{
  "task_type": "...",
  "selected_agent": "...",
  "model": "haiku|sonnet|opus",
  "phase": "discovery|architecture|implementation|qa|release",
  "fast_lane_assessment": {
    "eligible": true,
    "reason": "...",
    "recommended_skips": ["brainstorm"],
    "required_minimum_gates": ["verification"]
  },
  "objective": "...",
  "constraints": ["..."],
  "inputs": ["..."],
  "done_criteria": ["..."],
  "output_template": "template-id-from-agent-output-templates",
  "notes": "optional"
}
```

---

## 7. Suggested Execution Pattern (Mode B in Claude Code)

1. Brainstorm runs first → problem, scope, constraints, approved design direction.
2. Mode Selection Gate runs (see `modes.md`) → score, suggest mode, user approves.
3. Team Orchestrator receives brainstorm output and dispatches spec/plan leads.
4. Discovery Lead + Architecture Lead formalize the spec from brainstorm output (no re-brainstorm).
5. Orchestrator reviews spec internally → user approves spec.
6. Implementation Lead writes plan from approved spec.
7. Orchestrator reviews plan internally → user approves plan.
8. Phase lead dispatches stack implementer agents.
9. QA gate runs before phase completion.
10. DevOps gate runs before release decisions.
11. Orchestrator summarizes status, blockers, next actions.

Usage rule: route by task type using the dispatcher mapping first; every agent response must follow one template from `agent-output-templates.md` (prefer JSON for CI/review pipelines).
