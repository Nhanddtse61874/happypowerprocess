# Agent Output Templates (Standard Contracts)

Use these templates for all agent responses. Prefer JSON for automation and CI parsing.

## T0: Team Orchestrator
Template ID: `orchestrator-status-v1`

```json
{
  "template_id": "orchestrator-status-v1",
  "objective": "...",
  "current_phase": "discovery|architecture|implementation|qa|release",
  "ownership": [
    {
      "agent": "...",
      "task": "...",
      "status": "todo|in_progress|blocked|done"
    }
  ],
  "risks": [
    {
      "severity": "critical|important|suggestion",
      "description": "...",
      "mitigation": "..."
    }
  ],
  "next_actions": ["..."],
  "blockers": ["..."]
}
```

## T1: Discovery/Architecture/Implementation/Release Phase Lead
Template ID: `phase-lead-report-v1`

```json
{
  "template_id": "phase-lead-report-v1",
  "phase": "discovery|architecture|implementation|release",
  "summary": "...",
  "decisions": ["..."],
  "deliverables": ["..."],
  "open_questions": ["..."],
  "risks": [
    {
      "severity": "critical|important|suggestion",
      "description": "...",
      "owner": "..."
    }
  ],
  "done_criteria_result": [
    {
      "criterion": "...",
      "result": "pass|fail|partial",
      "evidence": "..."
    }
  ]
}
```

## T2: Implementer (All Language/Stack Implementers)
Template ID: `implementer-delivery-v1`

```json
{
  "template_id": "implementer-delivery-v1",
  "stack": "react-native|dotnet|angular|react|iot-edge",
  "task": "...",
  "changed_files": [
    {
      "path": "...",
      "change_type": "create|update|delete",
      "reason": "..."
    }
  ],
  "design_notes": ["..."],
  "tests": [
    {
      "name": "...",
      "type": "unit|integration|e2e|manual",
      "result": "pass|fail|not_run",
      "evidence": "..."
    }
  ],
  "quality_checks": {
    "lint": "pass|fail|not_run",
    "type_check": "pass|fail|not_run",
    "security_check": "pass|fail|not_run"
  },
  "risks": ["..."],
  "follow_ups": ["..."]
}
```

## T3: Specialist Role Report (React Native Senior, Backend Architect, Frontend Engineer, IoT Designer)
Template ID: `specialist-report-v1`

```json
{
  "template_id": "specialist-report-v1",
  "role": "senior-react-native-engineer|backend-architect-dotnet|frontend-engineer-angular|frontend-engineer-react|system-designer-mqtt-ble-iot",
  "context": "...",
  "recommendations": [
    {
      "priority": "critical|important|suggestion",
      "recommendation": "...",
      "rationale": "...",
      "impact": "..."
    }
  ],
  "tradeoffs": ["..."],
  "constraints_considered": ["..."],
  "next_steps": ["..."]
}
```

## T4: QA / Code Reviewer
Template ID: `qa-review-v1`

```json
{
  "template_id": "qa-review-v1",
  "scope": "...",
  "findings": [
    {
      "severity": "critical|important|suggestion",
      "title": "...",
      "description": "...",
      "location": "file[:line]",
      "recommendation": "..."
    }
  ],
  "coverage_assessment": {
    "requirements_coverage": "high|medium|low",
    "test_coverage_confidence": "high|medium|low",
    "regression_risk": "high|medium|low"
  },
  "release_recommendation": "approve|block|approve_with_conditions",
  "required_fixes": ["..."],
  "residual_risks": ["..."]
}
```

## T5: DevOps / CI-CD
Template ID: `devops-release-v1`

```json
{
  "template_id": "devops-release-v1",
  "pipeline_plan": ["..."],
  "quality_gates": ["..."],
  "deployment_strategy": "rolling|canary|blue_green|other",
  "rollback_plan": ["..."],
  "observability": {
    "logs": ["..."],
    "metrics": ["..."],
    "alerts": ["..."]
  },
  "release_readiness": "ready|not_ready|ready_with_conditions",
  "blocking_items": ["..."]
}
```

## T6: Fast Lane Assessment (Hotfix/Small Task)
Template ID: `fast-lane-assessment-v1`

```json
{
  "template_id": "fast-lane-assessment-v1",
  "eligible": true,
  "task_class": "hotfix|small_task|normal",
  "analysis": {
    "scope_is_explicit": true,
    "small_change_surface": true,
    "no_arch_or_api_change": true,
    "no_security_or_migration_risk": true,
    "low_regression_risk": true
  },
  "recommended_skips": ["brainstorm"],
  "required_minimum_gates": ["verification"],
  "reason": "...",
  "fallback_path": "normal_workflow_if_any_risk_changes"
}
```

## Minimal Human-Readable Block (Fallback)
If JSON is not possible, return this block:

```text
[TEMPLATE_ID] ...
Summary: ...
Key Outputs: ...
Risks: ...
Required Next Actions: ...
```

## Validation Rules
- Must include `template_id`.
- Must not omit required top-level keys for selected template.
- Use only allowed enums where defined.
- If unknown, return explicit `"not_run"`, `"partial"`, or empty list.
