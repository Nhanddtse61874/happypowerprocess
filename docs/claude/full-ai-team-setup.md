# Full AI Team Setup for Claude (Project-Agnostic)

## 1) Goal
Create a reusable AI engineering team setup so Claude can operate like a real multi-role software team across different projects.

Primary goals:
- Keep code clean, consistent, and easy to maintain
- Follow stack-specific best practices
- Self-review and detect defects early
- Support architecture and CI/CD design
- Keep behavior stable across sessions

Mode support:
- `Superpowers Solo Mode` (no team-agent dependency)
- `AI Team Spine Mode` (dispatcher + specialists + implementers)

## 2) Team Topology
Use one orchestrator plus specialist agents.

Core team:
- Team Orchestrator
- Master Dispatcher
- Senior React Native Engineer
- Backend Architect (.NET / C#)
- Frontend Engineer (Angular)
- Frontend Engineer (React)
- QA / Code Reviewer
- DevOps / CI-CD Assistant
- System Designer (MQTT / BLE / IoT)

## 3) Phase-Based Subagent Model
Run work in phases so each phase has a clear owner and output.

- Phase 1: Discovery and Scope
  Owner: `phase-discovery-lead`
  Output: clarified scope, constraints, acceptance criteria, risk list

- Phase 2: Architecture and Design
  Owner: `phase-architecture-lead`
  Output: architecture decisions, interfaces, trade-offs, rollout strategy

- Phase 3: Implementation
  Owner: `phase-implementation-lead`
  Output: incremental implementation with tests and change notes

- Phase 4: QA and Review Gate
  Owner: `phase-qa-gate`
  Output: severity-based findings, regression check, fix list

- Phase 5: Release and Operations
  Owner: `phase-release-devops-lead`
  Output: CI/CD plan, release checklist, rollback and observability plan

## 4) Language/Stack Implementer Agents
Use stack implementers for coding tasks.

- `implementer-react-native-typescript`
- `implementer-dotnet-csharp`
- `implementer-angular-typescript`
- `implementer-react-typescript`
- `implementer-iot-edge`

Each implementer must provide:
- What was changed
- Why this design was chosen
- Tests added or updated
- Risks and follow-up suggestions

Mandatory stack skills:
- `skills/implementer-dotnet-csharp/SKILL.md`
- `skills/implementer-react-native-typescript/SKILL.md`
- `skills/implementer-angular-typescript/SKILL.md`
- `skills/implementer-react-typescript/SKILL.md`
- `skills/implementer-iot-edge/SKILL.md`

## 5) Operating Rules
- Always start with constraints and acceptance criteria
- Prefer small, verifiable increments
- Enforce tests before declaring completion
- Separate architecture decisions from coding details
- Keep review findings prioritized: Critical, Important, Suggestion
- Never hide uncertainty; call out assumptions explicitly

## 6) Standard Handoff Contract
Every agent handoff should include:
- Context: problem, scope, constraints
- Input artifacts: files, APIs, related decisions
- Expected output: exact deliverables
- Done criteria: objective checks
- Risks: top known risks and mitigations

## 7) Suggested Execution Pattern in Claude Code
1. Team Orchestrator receives request and dispatches phase lead
2. Phase lead creates scoped tasks and dispatches specialist/implementer agents
3. QA gate runs before phase completion
4. DevOps gate runs before release decisions
5. Orchestrator summarizes status, blockers, and next actions

## 8) Master Dispatcher and Output Contracts
- Master dispatcher prompt: `docs/claude/master-dispatcher-prompt.md`
- Master dispatcher agent: `agents/master-dispatcher.md`
- Standard output templates: `docs/claude/agent-output-templates.md`
- End-to-end workflow runbook: `docs/claude/current-process-workflow.md`
- Runtime modes: `docs/claude/runtime-modes.md`
- Stack skill/rule map: `docs/claude/stack-skill-rule-map.md`
- Fast Lane support (hotfix/small task): `fast-lane-assessment-v1`

Usage rule:
- Route by task type using the dispatcher mapping first.
- Every agent response must follow one template from `agent-output-templates.md`.
- For CI/review pipelines, prefer the JSON format from each template.

## 9) Expected Benefits
- More predictable quality across sessions
- Faster delivery with clearer responsibilities
- Better architecture and operational readiness
- Lower regression rate due to built-in review gates
