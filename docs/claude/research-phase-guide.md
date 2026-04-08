# Research Phase Guide (Step 4)

Research phase chạy sau Mode Selection Gate và trước Spec Writing. Mục đích: thu thập evidence trước khi plan, thay vì plan từ assumption.

## Khi nào chạy

- Sau Step 3 (Mode Gate) — cả Mode A và Mode B đều chạy research
- Skip nếu: Fast Lane eligible, hoặc `workflow.research: false` trong config.json

## Critical Startup Protocol (mỗi research agent)

Trước khi research bất kỳ điều gì, agent phải:
1. Load CONTEXT.md nếu tồn tại — locked decisions ràng buộc scope
2. Load `.planning/config.json` — đọc validation settings
3. Đọc CLAUDE.md — project directives override mọi recommendation

## Mode A: Research (Solo) — 2 agents

| Agent | Nhiệm vụ | Output section |
|---|---|---|
| Stack Researcher | Patterns, conventions, best practices cho stack hiện tại | Stack Findings |
| Pitfall Researcher | Known issues, anti-patterns, gotchas với approach đang xem xét | Pitfalls & Anti-patterns |

## Mode B: Research (Team) — 4 agents + Synthesizer

| Agent | Nhiệm vụ | Output file |
|---|---|---|
| Stack Researcher | Current stack với versions và rationale | `.planning/research/STACK.md` |
| Feature Researcher | Table stakes vs differentiators vs anti-features | `.planning/research/FEATURES.md` |
| Architecture Researcher | Component boundaries, data flow, build order | `.planning/research/ARCHITECTURE.md` |
| Pitfall Researcher | Common mistakes, prevention strategies, edge cases | `.planning/research/PITFALLS.md` |
| **Research Synthesizer** | Consolidate 4 outputs thành actionable summary | `.planning/research/SUMMARY.md` |

**Synthesizer chạy sau 4 agents** — SUMMARY.md là input chính cho Spec Writing (Step 5).

## Claim Provenance (CRITICAL)

Mọi factual claim phải được tag nguồn:

| Tag | Ý nghĩa | Trust level |
|---|---|---|
| `[VERIFIED: npm registry]` | Tool-confirmed | HIGH |
| `[CITED: docs.example.com]` | Official documentation | HIGH-MEDIUM |
| `[ASSUMED]` | Training knowledge, cần user validation | LOW |

**Không bao giờ present assumed knowledge như verified fact** — đặc biệt với compliance, security, performance.

**Honest research philosophy:**
- Verify trước khi assert — training data có thể 6-18 tháng cũ
- Report gaps honestly — "couldn't find X" có giá trị hơn fabricate
- Flag LOW confidence rõ ràng — surfaces items cần validation
- Treat evidence as driver, not confirmation bias

## Tool Priority cho Research

1. **Context7** — Library APIs, configs, versions (HIGH trust)
2. **Official docs / WebFetch** — READMEs, changelogs (HIGH-MEDIUM)
3. **WebSearch** — Ecosystem patterns (cần cross-verify)

## Research Workflow (từng agent)

| Step | Action |
|------|--------|
| 1 | Load phase context, CONTEXT.md, project constraints |
| 2 | Identify research domains |
| 2.5 | *Rename/refactor phases only:* Runtime state inventory (stored data, live config, OS registration, secrets, build artifacts) |
| 2.6 | *External dependency phases:* Environment availability audit via bash probes |
| 3 | Execute research theo tool priority |
| 4 | Map requirements to tests, identify gaps (nếu enabled) |
| 5 | Quality check: domains covered, negatives verified, confidence assigned |
| 6 | Write output file |

## RESEARCH.md Structure (Mode A output)

```markdown
# {Phase} Research

**Date:** {YYYY-MM-DD}
**Mode:** A
**Input:** Brainstorm output + approved design direction

<user_constraints>
## User Constraints (from CONTEXT.md, nếu tồn tại)
### Locked Decisions
### Claude's Discretion
### Deferred Ideas (OUT OF SCOPE)
</user_constraints>

<phase_requirements>
## Phase Requirements
[Map REQ-IDs to research findings]
</phase_requirements>

## Standard Stack

{Stack findings với claim tags}

## Architecture Patterns

{Pattern findings}

## Don't Hand-Roll

{Libraries/tools đã có, không tự viết lại}

## Common Pitfalls

{Anti-patterns cần tránh}

## Assumptions Log

{Mọi ASSUMED claim — cần user validation}

## Open Questions

{Những gì không tìm thấy hoặc cần clarification}

## Sources

{URLs, docs tham khảo}
```

## Human Touchpoint

Sau khi research xong: User review `.planning/{phase}-RESEARCH.md` (Mode A) hoặc `.planning/research/SUMMARY.md` (Mode B), confirm đủ thông tin trước khi tiếp tục Step 5.

Nếu thiếu: thêm targeted research agent cho vùng cụ thể, không cần restart toàn bộ phase.

## Dispatcher Routing

| Task type | Owner |
|---|---|
| `research-phase-mode-a` | Main session (2 parallel subagents) |
| `research-phase-mode-b` | `team-orchestrator` (4 parallel subagents + synthesizer) |
