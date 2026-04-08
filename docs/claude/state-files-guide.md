# State Files Guide

State files là "bộ nhớ thực" của project — sống trong repo, đọc được bởi AI và human, persist qua mọi context reset.

## Cấu trúc

```
{project-root}/
├── PROJECT.md          — Vision document, không thay đổi trong milestone
├── REQUIREMENTS.md     — Scope v1/v2 với phase traceability
├── ROADMAP.md          — Milestone list + tiến độ
├── STATE.md            — Đang ở đâu, quyết định gì, blockers
└── .planning/
    ├── {phase}-RESEARCH.md      — Research output per phase
    ├── {phase}-{N}-PLAN.md      — Plan per task group (XML structure)
    ├── {phase}-SUMMARY.md       — Ghi lại những gì đã xảy ra
    └── {phase}-UAT.md           — User acceptance test results
```

## Khi nào tạo

- **New project (Step 0)**: Tạo PROJECT.md + REQUIREMENTS.md + ROADMAP.md + STATE.md
- **Brownfield project (Step 0)**: Map codebase trước → tạo state files từ kết quả mapping
- **Resuming session**: Đọc STATE.md → biết ngay phase, blockers, next action

## Khi nào cập nhật

| File | Cập nhật khi |
|---|---|
| PROJECT.md | Rất hiếm — chỉ khi vision thay đổi căn bản |
| REQUIREMENTS.md | Brainstorm output mới, scope thay đổi |
| ROADMAP.md | Milestone complete, milestone mới bắt đầu |
| STATE.md | Sau mỗi step hoàn thành, khi có blocker mới |
| {phase}-RESEARCH.md | Sau Step 4 (Research) |
| {phase}-{N}-PLAN.md | Sau Step 6 (Planning) |
| {phase}-SUMMARY.md | Sau Step 11 (Ship) |
| {phase}-UAT.md | Sau Step 8 (UAT) |

## Templates

Xem `docs/claude/templates/` cho template của từng file.

## Nguyên tắc

- STATE.md là nguồn truth duy nhất cho "đang ở đâu"
- Không ghi code patterns, git history, hay debug solutions vào state files
- Dùng absolute dates không phải relative ("2026-04-08" không phải "hôm nay")
- STATE.md phải đủ để session mới đọc vào và biết chính xác phải làm gì tiếp theo
