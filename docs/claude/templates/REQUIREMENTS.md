# Requirements

**Source:** Brainstorm output {YYYY-MM-DD}
**Phase traceability:** Step 2 (Brainstorm) → Step 5 (Spec)

## REQ-ID Format

`[CATEGORY]-[NUMBER]` — ví dụ: `AUTH-01`, `CONT-02`, `UI-01`

Mỗi requirement phải:
- **Specific & testable**: "User can reset password via email link" (không phải "add auth")
- **User-centric**: "User can X" (không phải "system does X")
- **Atomic**: Một capability per requirement
- **Independent**: Minimal cross-dependencies

Mọi v1 requirement phải map về đúng một phase trong ROADMAP.md — 100% coverage bắt buộc.

## v1 Requirements (Ship in initial release)

| REQ-ID | Requirement | Phase |
|---|---|---|
| {CAT}-01 | User can {specific, testable action} | Phase {N} |
| {CAT}-02 | User can {specific, testable action} | Phase {N} |

## v2 Requirements (Deferred — table stakes users expect)

| REQ-ID | Requirement | Reason deferred |
|---|---|---|
| {CAT}-03 | User can {action} | {why v2} |

## Out of Scope (Explicit exclusions)

| Item | Reason |
|---|---|
| {Feature} | {Reasoning} |

## Assumptions

- {Assumption 1}

## Last updated

{YYYY-MM-DD}
