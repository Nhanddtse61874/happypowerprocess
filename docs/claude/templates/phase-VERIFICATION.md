# {Phase Name} — Verification

**Phase:** {Step N}
**Date:** {YYYY-MM-DD}
**Method:** Goal-backward analysis

## Goal-Backward Analysis

Starting from phase goal, verified what MUST be true:

### Observable Truths (User perspective)

| # | Truth | Status | Evidence |
|---|---|---|---|
| T-1 | User can {observable behavior} | PASS / FAIL / PARTIAL | {where verified} |
| T-2 | User can {observable behavior} | PASS / FAIL / PARTIAL | {where verified} |

### Required Artifacts

| Artifact | Exists? | Notes |
|---|---|---|
| `{path/to/file}` | YES / NO | {note} |

### Required Wiring (Critical connections)

| Connection | Verified? | Notes |
|---|---|---|
| {Component A} → {Component B} | YES / NO | {note} |

## Requirement Coverage

| REQ-ID | Requirement | Status | Evidence |
|---|---|---|---|
| {CAT}-01 | {requirement} | PASS / FAIL | {evidence} |

## Gaps Found

| # | Gap | Severity | Suggested fix |
|---|---|---|---|
| G-1 | {description} | critical/important/minor | {fix approach} |

## Overall Result

**PASS / FAIL / PARTIAL**

## Decision

- [ ] PASS → proceed to next step
- [ ] PARTIAL → run gap closure (`/gsd-plan-phase {N} --gaps`)
- [ ] FAIL → back to Execute (Step 7) with fix plans
