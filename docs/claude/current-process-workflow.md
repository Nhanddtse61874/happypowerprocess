# Unified Workflow: GSD + Superpowers (v2)

Workflow kết hợp GSD state management, research phase, wave-based execution, UAT gate, và milestone rhythm với Superpowers Mode A/B, Fast Lane, QA gate, DevOps phase, và escalation protocol.

**Nguyên tắc:** AI đồng hành qua từng step — không tự động chạy step tiếp theo mà không có human approval.

Reference files:
- State files: `docs/claude/state-files-guide.md`
- Templates: `docs/claude/templates/`
- Research phase: `docs/claude/research-phase-guide.md`
- Mode selection: `docs/claude/mode-selection-criteria.md`
- Agent templates: `docs/claude/agent-output-templates.md`
- Stack skill map: `docs/claude/stack-skill-rule-map.md`

---

## STEP 0 — Project Bootstrap

**Làm gì:** Thiết lập hoặc đọc project state để biết context đầy đủ trước khi làm bất kỳ điều gì.

**New project:**
- Tạo PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md từ templates trong `docs/claude/templates/`
- Tạo thư mục `.planning/`

**Brownfield project:**
- Map codebase: phân tích stack, conventions, architecture patterns hiện tại
- Tạo state files từ kết quả mapping

**Resuming session:**
- Đọc STATE.md → biết ngay: đang ở Step nào, next action là gì, blockers là gì
- Tiếp tục từ step đó, không restart

**Human touchpoint:** User xác nhận project context trước khi tiếp tục.

**Output:** PROJECT.md, REQUIREMENTS.md, ROADMAP.md, STATE.md (created hoặc read)

---

## STEP 1 — Fast Lane Check

**Làm gì:** Đánh giá task có đủ điều kiện Fast Lane không.

**Fast Lane criteria (tất cả phải đúng):**
- Fix scope rõ ràng, không ambiguous
- Thay đổi nhỏ (1-2 files)
- Không có architecture/API contract change
- Không có security hoặc data migration impact
- Regression risk thấp

**Nếu eligible:** Skip Steps 2-4, vào thẳng Step 5 (Spec) với Fast Lane output làm input
**Nếu not eligible:** Tiếp tục Step 2

**Template output:** `fast-lane-assessment-v1`

**Human touchpoint:** User confirm Fast Lane hoặc full workflow.

---

## STEP 2 — Brainstorm

**Làm gì:** Hiểu rõ yêu cầu, scope, constraints. Propose 2-3 approaches với trade-offs.

**Skill:** `skills/brainstorming/SKILL.md`

**Output:**
- Problem statement, scope, constraints, success criteria
- 2-3 approaches với trade-offs
- Approved design direction
- REQUIREMENTS.md được cập nhật từ brainstorm output

**Human touchpoint:** User approve design direction. Không tiếp tục nếu chưa approve.

---

## STEP 3 — Mode Selection Gate

**Làm gì:** Score 5 criteria từ brainstorm output, suggest Mode A hoặc B.

**Reference:** `docs/claude/mode-selection-criteria.md`

**Scoring criteria:**
1. Domain count (1 vs 2+)
2. Risk level (low/medium/high)
3. QA/DevOps gate needed (yes/no)
4. Cross-team coordination (yes/no)
5. Output format (formal/informal)

**Threshold:** 0-1 B signals → Mode A | 2 B signals → Mode A (Mode B viable) | 3+ → Mode B

**Mode A:** Solo, nhẹ, 1 domain, không cần formal QA gate
**Mode B:** AI team spine, multi-domain, cần QA/DevOps formal, role-based ownership

**Human touchpoint:** User approve mode. Sau đây mode không thay đổi nữa — đây là source of truth.

---

## STEP 4 — Research

**Làm gì:** Parallel research agents thu thập evidence trước khi viết spec/plan.

**Reference:** `docs/claude/research-phase-guide.md`

**Mode A:** 2 parallel agents (Stack Researcher + Pitfall Researcher)
**Mode B:** 4 parallel agents (Stack + Architecture + Feature + Pitfall)

**Output:** `.planning/{phase}-RESEARCH.md`

**Human touchpoint:** User review research findings, confirm đủ trước khi tiếp tục Step 5.
Nếu thiếu: thêm targeted research agent, không cần restart phase.

**Skip nếu:** Fast Lane eligible.

---

## STEP 5 — Spec Writing

**Làm gì:** Viết technical spec từ brainstorm output + research findings.

**Mode A:** `skills/brainstorming/SKILL.md` — solo spec writing
**Mode B:**
- `agents/phase-discovery-lead.md`: formalize requirements spec từ brainstorm output
- `agents/phase-architecture-lead.md`: formalize technical spec (contracts, interfaces, trade-offs)
- Input: brainstorm output + research findings (không re-brainstorm)

**Output:** `docs/superpowers/specs/YYYY-MM-DD-{topic}-design.md`

**Mode B output templates:** `phase-lead-report-v1`

**Human touchpoint:** User approve spec. Nếu cần sửa → quay lại spec, không brainstorm lại.

---

## STEP 6 — Planning

**Làm gì:** Viết implementation plan dạng XML tasks, nhóm theo waves.

**Mode A:** `skills/writing-plans/SKILL.md`
**Mode B:** `agents/phase-implementation-lead.md`

**Task structure:** XML format — xem `docs/claude/templates/plan-task.xml`

**Wave grouping:**
- Tasks độc lập (không phụ thuộc nhau) → cùng wave, chạy song song
- Tasks phụ thuộc vào wave trước → wave sau

**Output:** `.planning/{phase}-{N}-PLAN.md`

**Human touchpoint:** User approve plan + wave structure trước khi execute. Nếu cần sửa → edit plan, không rewrite spec.

---

## STEP 7 — Execution (Wave by Wave)

**Làm gì:** Execute từng wave, mỗi task trong fresh context độc lập.

**Skill:** `skills/subagent-driven-development/SKILL.md` hoặc `skills/executing-plans/SKILL.md`

**Cơ chế:**
- Cùng wave → tasks chạy song song (subagent per task)
- Fresh context per task: mỗi subagent chỉ nhận đúng files cần thiết cho task đó
- Stack skill mandatory: dotnet/react/react-native/angular/iot-edge theo `docs/claude/stack-skill-rule-map.md`
- Atomic commit: task xong → commit ngay (`feat({phase}): {task name}`)

**Mode B:** Stack implementer agents được dispatcher chọn:
- `agents/implementer-react-native-typescript.md`
- `agents/implementer-dotnet-csharp.md`
- `agents/implementer-angular-typescript.md`
- `agents/implementer-react-typescript.md`
- `agents/implementer-iot-edge.md`
- Output: `implementer-delivery-v1`

**Human touchpoint:** User approve kết quả mỗi wave trước khi chạy wave tiếp theo.

---

## STEP 8 — UAT (User Acceptance Testing)

**Làm gì:** User tự test feature theo acceptance criteria. AI không tự claim "done".

**Cơ chế:**
- AI tạo `.planning/{phase}-UAT.md` từ template (`docs/claude/templates/phase-UAT.md`)
- User chạy feature, test từng AC, ghi kết quả
- AI không tự verify thay user — đây là human step

**Pass:** Tiếp tục Step 9 (QA Gate)
**Fail:** AI tạo fix plan → back to Step 7 (không replan toàn bộ, chỉ fix tasks cụ thể)

**Template output:** `uat-gate-v1`

**Human touchpoint:** Đây toàn bộ là human step. AI hỗ trợ tạo fix plan nếu cần.

---

## STEP 9 — QA Gate

**Làm gì:** Formal QA review với severity classification.

**Mode A:** `skills/requesting-code-review/SKILL.md`
**Mode B:** `agents/phase-qa-gate.md` + `agents/qa-code-reviewer.md`

**Findings severity:** Critical | Important | Suggestion

**Block:** Fix → back to Step 7 với targeted fix plan
**Approve / Approve with conditions:** Tiếp tục Step 10

**Template output:** `qa-review-v1`

**Human touchpoint:** User review QA findings, quyết định fix hay approve-with-conditions.

---

## STEP 10 — Release / DevOps

**Làm gì:** CI/CD plan, deployment strategy, rollback, observability.

**Mode A:** `skills/finishing-a-development-branch/SKILL.md`
**Mode B:** `agents/phase-release-devops-lead.md` + `agents/devops-cicd-assistant.md`

**Template output (Mode B):** `devops-release-v1`

**Human touchpoint:** User approve release readiness trước khi ship.

**Skip nếu:** Mode A và task không cần formal release gate.

---

## STEP 11 — Ship & Milestone Close

**Làm gì:** Merge/PR + cập nhật state files + mở milestone mới nếu cần.

**Cơ chế:**
1. Tạo PR hoặc merge theo strategy từ Step 10
2. Viết `.planning/{phase}-SUMMARY.md` từ template (`docs/claude/templates/phase-SUMMARY.md`)
3. Cập nhật ROADMAP.md: milestone này done
4. Cập nhật STATE.md: current position = start of next milestone (hoặc project complete)
5. Escalation: nếu có risk/conflict chưa resolve → 2-3 options cho user quyết định

**Template output:** `phase-summary-v1`
**Mode B:** `agents/team-orchestrator.md` tổng hợp final handoff (`orchestrator-status-v1`)

**Human touchpoint:** User quyết định ship, review summary, quyết định next milestone.

---

## Fast Lane Path

```
STEP 0 (Bootstrap) → STEP 1 (Fast Lane: eligible) → STEP 5 (Spec) → STEP 6 (Plan)
→ STEP 7 (Execute) → STEP 8 (UAT) → STEP 9 (QA) → STEP 11 (Ship)
```

Skip: Steps 2, 3, 4, 10 (trừ khi Mode B và cần formal release gate)

---

## Full Path Summary

```
STEP 0:  Bootstrap        → đọc/tạo state files                [USER CONFIRMS]
STEP 1:  Fast Lane?       → eligible: skip to STEP 5           [USER CONFIRMS]
STEP 2:  Brainstorm       → approved design direction           [USER APPROVES]
STEP 3:  Mode Gate        → Mode A hoặc B                      [USER APPROVES]
STEP 4:  Research         → parallel agents, RESEARCH.md        [USER REVIEWS]
STEP 5:  Spec             → solo (A) / team agents (B)          [USER APPROVES]
STEP 6:  Plan             → XML tasks + wave grouping           [USER APPROVES]
STEP 7:  Execute          → wave by wave, fresh context         [USER APPROVES each wave]
STEP 8:  UAT              → user tests feature                  [USER DRIVES]
STEP 9:  QA Gate          → severity findings + fix loop        [USER APPROVES]
STEP 10: Release/DevOps   → CI/CD plan (Mode B)                [USER APPROVES]
STEP 11: Ship             → PR/merge + summary + next milestone [USER DECIDES]
```

---

## Output Standards

- Mode A: Superpowers skill outputs
- Mode B: template_id từ `docs/claude/agent-output-templates.md` (bắt buộc)
- Cả hai: atomic commits sau mỗi task trong Step 7
- Cả hai: state files luôn cập nhật sau mỗi step hoàn thành
