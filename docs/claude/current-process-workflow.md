# Unified Workflow: GSD + Superpowers (v3)

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

**Artifact persistence principle:** Mỗi artifact phải được viết ra disk ngay lập tức sau khi tạo — nếu context bị mất, artifacts vẫn còn và có thể resume.

### New project

1. Detect greenfield vs brownfield (có `.git` chưa, có codebase chưa)
2. Setup git nếu cần
3. Thu thập config upfront — lưu vào `.planning/config.json`:
   - **mode**: `interactive` (dừng confirm từng step) hoặc `yolo` (auto-approve)
   - **granularity**: `coarse` (3-5 phases) / `standard` (5-8) / `fine` (8-12+)
   - **parallelization**: `true` / `false`
   - **commit_docs**: commit planning artifacts vào git không
   - **model_profile**: `balanced` / `quality` / `budget`
   - **workflow.research**: bật research phase không
   - **workflow.plan_check**: bật plan checker không
4. **Deep questioning** (interactive mode): hỏi freeform về project vision — không dùng checklist, follow conversational threads tự nhiên. Probe:
   - Điều gì khiến họ excited về idea này
   - Problem cụ thể nào đang giải quyết
   - Vague term clarifications với concrete examples
   - Existing decisions đã có
   - Decision gate: khi context đủ → tạo PROJECT.md
5. Tạo state files từ templates (`docs/claude/templates/`):
   - `PROJECT.md`, `REQUIREMENTS.md` với REQ-IDs, `ROADMAP.md`, `STATE.md`
   - Tạo thư mục `.planning/` và `.planning/research/`
6. Commit artifacts ngay: `git commit -m "init: project state files"`

### Brownfield project

1. Map codebase: phân tích stack, conventions, architecture patterns hiện tại
2. Runtime state inventory nếu có migration: stored data, live config, OS registration, secrets, build artifacts
3. Tạo state files từ kết quả mapping

### Resuming session

- Đọc `STATE.md` → biết ngay: đang ở Step nào, next action là gì, blockers là gì
- Tiếp tục từ step đó, không restart

**Human touchpoint:** User xác nhận project context và config trước khi tiếp tục.

**Output:** `.planning/config.json`, `PROJECT.md`, `REQUIREMENTS.md` (với REQ-IDs), `ROADMAP.md`, `STATE.md`

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
- REQUIREMENTS.md được cập nhật — requirements phải có REQ-IDs, testable, user-centric, atomic

**REQ-ID rule:** Mọi v1 requirement phải map về đúng một phase trong ROADMAP.md. 100% coverage bắt buộc.

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

**Mode A:** Solo, nhẹ, 1 domain
**Mode B:** AI team spine, multi-domain, cần formal QA/DevOps gate, role-based ownership

**Human touchpoint:** User approve mode. Sau đây mode không thay đổi nữa — đây là source of truth.

---

## STEP 4 — Research

**Làm gì:** Parallel research agents thu thập evidence trước khi viết spec. Plans phải dựa trên evidence, không phải assumption.

**Reference:** `docs/claude/research-phase-guide.md`

**Skip nếu:** Fast Lane eligible hoặc `workflow.research: false` trong config.

### Critical startup (mỗi agent)

Trước khi research:
1. Load CONTEXT.md — locked decisions ràng buộc scope
2. Load `.planning/config.json` — validation settings
3. Đọc CLAUDE.md — project directives override recommendations

### Mode A: 2 parallel agents

| Agent | Output |
|---|---|
| Stack Researcher | Stack findings với [VERIFIED/CITED/ASSUMED] tags |
| Pitfall Researcher | Anti-patterns, gotchas, common mistakes |

**Output:** `.planning/{phase}-RESEARCH.md`

### Mode B: 4 parallel agents + Synthesizer

| Agent | Output file |
|---|---|
| Stack Researcher | `.planning/research/STACK.md` |
| Feature Researcher | `.planning/research/FEATURES.md` |
| Architecture Researcher | `.planning/research/ARCHITECTURE.md` |
| Pitfall Researcher | `.planning/research/PITFALLS.md` |
| **Research Synthesizer** (chạy sau) | `.planning/research/SUMMARY.md` |

**Claim provenance (bắt buộc):** Mọi claim phải có tag `[VERIFIED]`, `[CITED]`, hoặc `[ASSUMED]`. Không present assumed knowledge như fact.

**Human touchpoint:** User review research output, confirm đủ trước khi tiếp tục Step 5.
Nếu thiếu: thêm targeted research agent, không restart toàn phase.

---

## STEP 5 — Spec Writing

**Làm gì:** Viết technical spec từ brainstorm output + research findings.

**Mode A:** `skills/brainstorming/SKILL.md` — solo spec writing
**Mode B:**
- `agents/phase-discovery-lead.md`: formalize requirements spec từ brainstorm output + RESEARCH.md
- `agents/phase-architecture-lead.md`: formalize technical spec (contracts, interfaces, trade-offs) từ brainstorm + research
- Input: brainstorm output + research findings (không re-brainstorm)
- Output templates: `phase-lead-report-v1`

**Output:** `docs/superpowers/specs/YYYY-MM-DD-{topic}-design.md`

**Human touchpoint:** User approve spec. Nếu cần sửa → quay lại spec, không brainstorm lại.

---

## STEP 6 — Planning

**Làm gì:** Viết implementation plan với goal-backward methodology, XML tasks nhóm theo waves.

**Mode A:** `skills/writing-plans/SKILL.md`
**Mode B:** `agents/phase-implementation-lead.md`

### Goal-Backward Methodology (bắt buộc)

Trước khi viết tasks, derive từ phase goal:

1. **Observable Truths** (3-7, user perspective): "User can see messages", "User can send message"
2. **Required Artifacts** (specific files): `src/components/Chat.tsx`, `src/api/chat/route.ts`
3. **Required Wiring** (connections): Chat component fetches từ /api/chat on mount
4. **Key Links** (critical failure points): Where breakage cascades

→ Output vào `must_haves` frontmatter của PLAN.md

### Task Structure

Xem `docs/claude/templates/plan-task.xml` cho full template.

Mỗi task bắt buộc có:
- `<read_first>`: Files executor phải đọc trước khi bắt đầu
- `<action>`: Specific implementation — include WHAT to AVOID và why, REQ-ID tương ứng
- `<verify><automated>`: Command chạy <60 giây (Nyquist Rule)
- `<done>`: Measurable, grep-verifiable criteria

### Context Budget Rule

- Target ~50% token usage per plan
- Maximum 2-3 tasks per plan
- Nếu scope exceeds budget → split thành smaller plans, **không bao giờ nén**

### Wave Assignment Algorithm

```
if task.depends_on is empty → Wave 1
else → Wave = max(wave of dependencies) + 1
if task.files_modified overlap with earlier task → force later wave
```

Same-wave plans phải có **zero file overlap** — bắt buộc.

### Plan Checker (nếu `workflow.plan_check: true`)

Sau khi plan viết xong, Plan Checker validate 11 dimensions trước khi approve:

| # | Dimension | Question |
|---|---|---|
| 1 | Requirement Coverage | Mọi REQ-ID có covering task? |
| 2 | Task Completeness | Mọi task có read_first + action + verify + done? |
| 3 | Dependency Correctness | Dependencies valid, acyclic, đúng wave? |
| 4 | Key Links Planned | Artifacts wired together (không chỉ created)? |
| 5 | Scope Sanity | 2-3 tasks/plan, ~50% context? |
| 6 | Verification Derivation | must_haves trace về phase goal? |
| 7 | Context Compliance | Plans honor CONTEXT.md decisions? |
| 8 | Nyquist Compliance | Automated verify present và <60s? |
| 9 | Cross-Plan Data Contracts | Shared data transforms compatible? |
| 10 | CLAUDE.md Compliance | Plans respect project conventions? |
| 11 | Research Resolution | Open questions đã answered? |

**Scope Reduction Detection:** Nếu plan reference user decisions nhưng deliver "v1 stubs" → **always blocker**.

**Revision loop:** Max 3 iterations. Nếu issue count không giảm → escalate cho user.

**Output:** `.planning/{phase}-{N}-PLAN.md`

**Human touchpoint:** User approve plan + wave structure trước khi execute.

---

## STEP 7 — Execution (Wave by Wave)

**Làm gì:** Execute từng wave, mỗi task trong fresh context độc lập.

**Skill:** `skills/subagent-driven-development/SKILL.md` hoặc `skills/executing-plans/SKILL.md`

### Intra-wave Safety Check

Trước khi execute một wave:
- Kiểm tra file overlap giữa các plans trong cùng wave
- Nếu 2 plans modify cùng file → force sequential, không parallel

### Fresh Context per Task

- Mỗi subagent chỉ nhận **đúng files cần thiết** từ `<read_first>` — không nhận toàn bộ codebase
- Fresh 200k-token context per task — tránh context rot hoàn toàn

### Worktree Isolation

Khi `parallelization: true`:
- Mỗi executor chạy trong isolated git worktree riêng
- Worktree creation phải **sequential** (tránh `.git/config.lock` race condition)
- Sau wave: merge worktree branches về main
- **File protection**: Restore STATE.md, ROADMAP.md từ main branch sau merge — tránh stale version overwrite

### Atomic Commits

Task xong → commit **ngay lập tức**:
```
feat({phase}): {task name}
```
Không batch commits. Mỗi task là một commit độc lập — bisectable, revertable.

### Stack Skill (bắt buộc cả hai mode)

Theo `docs/claude/stack-skill-rule-map.md`:
- `.NET/C#` → `skills/implementer-dotnet-csharp/SKILL.md`
- `React Native` → `skills/implementer-react-native-typescript/SKILL.md`
- `Angular` → `skills/implementer-angular-typescript/SKILL.md`
- `React` → `skills/implementer-react-typescript/SKILL.md`
- `IoT/MQTT/BLE` → `skills/implementer-iot-edge/SKILL.md`

### Mode B Stack Agents

Dispatcher chọn implementer agent theo stack:
- `agents/implementer-react-native-typescript.md`
- `agents/implementer-dotnet-csharp.md`
- `agents/implementer-angular-typescript.md`
- `agents/implementer-react-typescript.md`
- `agents/implementer-iot-edge.md`
- Output: `implementer-delivery-v1`

### Failure Recovery

Nếu executor fail (missing SUMMARY.md, test fail):
- Report plan nào failed
- User quyết định: retry / skip / abort
- Partial progress ghi vào STATE.md để resume

**Human touchpoint:** User approve kết quả mỗi wave trước khi chạy wave tiếp theo.

---

## STEP 8 — UAT + Verification

**Hai phần riêng biệt:**

### 8a — UAT (User Acceptance Testing)

**Làm gì:** User tự test feature theo acceptance criteria. AI không tự claim "done".

- AI tạo `.planning/{phase}-UAT.md` từ template (`docs/claude/templates/phase-UAT.md`)
- User chạy feature, test từng AC, ghi kết quả bằng plain text
- **Show expected, ask if reality matches** — AI mô tả expected behavior, user confirm hoặc mô tả sự khác biệt
- AI infer severity từ mô tả của user — không questionnaire

**Pass:** Tiếp tục 8b
**Fail:** AI spawn parallel debug agents → diagnose root causes → gsd-planner tạo fix plans → Plan Checker verify (max 3 revision cycles) → back to Step 7

**Template output:** `uat-gate-v1`

### 8b — Goal-Backward Verification

**Làm gì:** AI cross-reference implementation artifacts với phase must_haves.

- Extract success criteria từ phase goal trong ROADMAP.md
- Cross-reference với SUMMARY.md files từ execution
- Tạo `.planning/{phase}-VERIFICATION.md` (xem `docs/claude/templates/phase-VERIFICATION.md`)

**Gaps found:** Offer gap closure → `/gsd-plan-phase {N} --gaps` tạo gap-closure plans → back to Step 7

**Regression gate (Mode B):** Chạy prior phase test suites để catch cross-phase regressions.

**Schema drift detection (nếu relevant):** Verify TypeScript build pass nhưng DB schema vẫn in sync.

**Human touchpoint:** Đây toàn bộ là human step cho UAT. Verification AI hỗ trợ nhưng user quyết định final.

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
2. Viết `.planning/{phase}-SUMMARY.md` từ template — ghi: đã làm gì, quyết định gì, commits, lessons, residual risks
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
→ STEP 7 (Execute) → STEP 8 (UAT + Verification) → STEP 9 (QA) → STEP 11 (Ship)
```

Skip: Steps 2, 3, 4, 10 (trừ khi Mode B và cần formal release gate)

---

## Full Path Summary

```
STEP 0:  Bootstrap          → config.json, state files, deep questioning  [USER CONFIRMS]
STEP 1:  Fast Lane?         → eligible: skip to STEP 5                    [USER CONFIRMS]
STEP 2:  Brainstorm         → approved direction, REQ-IDs updated          [USER APPROVES]
STEP 3:  Mode Gate          → Mode A hoặc B                               [USER APPROVES]
STEP 4:  Research           → parallel agents + synthesizer, claim tags    [USER REVIEWS]
STEP 5:  Spec               → solo (A) / team agents (B)                  [USER APPROVES]
STEP 6:  Plan               → goal-backward, XML tasks, wave groups        [USER APPROVES]
          └─ Plan Checker   → 11-dimension validation, max 3 revision loops
STEP 7:  Execute            → wave by wave, fresh context, atomic commits  [USER APPROVES each wave]
          └─ Intra-wave overlap check, worktree isolation, failure recovery
STEP 8:  UAT + Verify       → user tests + goal-backward verification      [USER DRIVES]
          └─ Gap closure loop, regression gate, schema drift detection
STEP 9:  QA Gate            → severity findings + fix loop                 [USER APPROVES]
STEP 10: Release/DevOps     → CI/CD plan (Mode B)                         [USER APPROVES]
STEP 11: Ship               → PR/merge + SUMMARY + ROADMAP + STATE updated [USER DECIDES]
```

---

## Output Standards

- Mode A: Superpowers skill outputs
- Mode B: `template_id` từ `docs/claude/agent-output-templates.md` (bắt buộc)
- Cả hai: atomic commits sau mỗi task trong Step 7
- Cả hai: artifact persistence — viết ra disk ngay lập tức, không giữ trong memory
- Cả hai: state files cập nhật sau mỗi step hoàn thành
- REQ-ID traceability: mọi v1 requirement phải map về ít nhất một plan task
