# Research Phase Guide (Step 4)

Research phase chạy sau Mode Selection Gate và trước Spec Writing. Mục đích: thu thập evidence trước khi plan, thay vì plan từ assumption.

## Khi nào chạy

- Sau Step 3 (Mode Gate) — cả Mode A và Mode B đều chạy research
- Skip nếu: Fast Lane eligible (task đã rõ ràng, không cần research)

## Mode A: Research (Solo)

Chạy 2 parallel research agents:

| Agent | Nhiệm vụ | Output section |
|---|---|---|
| Stack Researcher | Patterns, conventions, best practices cho stack hiện tại | Stack Findings |
| Pitfall Researcher | Known issues, anti-patterns, gotchas với approach đang xem xét | Pitfalls & Anti-patterns |

## Mode B: Research (Team)

Chạy 4 parallel research agents:

| Agent | Nhiệm vụ | Output section |
|---|---|---|
| Stack Researcher | Patterns, conventions, best practices | Stack Findings |
| Architecture Researcher | Architecture options, trade-offs, scalability | Architecture Findings |
| Feature Researcher | Implementation approaches cho từng feature trong scope | Feature Approach Findings |
| Pitfall Researcher | Anti-patterns, known failures, edge cases | Pitfalls & Anti-patterns |

## Output Format

Lưu vào `.planning/{phase}-RESEARCH.md`:

```markdown
# {Phase} Research

**Date:** {YYYY-MM-DD}
**Mode:** {A / B}
**Input:** Brainstorm output + approved design direction

## Stack Findings

{Patterns, conventions, libraries phù hợp}

## Architecture Findings (Mode B only)

{Options, trade-offs, recommendation}

## Feature Approach Findings (Mode B only)

{Implementation approach cho từng feature}

## Pitfalls & Anti-patterns

{Known issues, gotchas, cần tránh}

## Recommendation Summary

{Một đoạn tổng hợp — đây là input cho Spec Writing (Step 5)}
```

## Human Touchpoint

Sau khi research xong: User review `.planning/{phase}-RESEARCH.md`, confirm đủ thông tin trước khi tiếp tục Step 5 (Spec).

Nếu thiếu: thêm targeted research agent cho vùng cụ thể, không cần restart toàn bộ phase.

## Dispatcher Routing

Task type: `research-phase-mode-a` hoặc `research-phase-mode-b`
Owner: Orchestrated by main session (Mode A) hoặc team-orchestrator (Mode B)
