# Knowledge-Mining Pilot Protocol

**Date:** 2026-07-07
**Status:** PILOT — not yet a plugin skill. Run once on a real target project, capture field notes, then formalize into `skills/mining-project-knowledge/` (see §9).
**Origin:** External "skill library authoring" spec (Phase 1 Discover → Phase 2 Author → Phase 3 Review), merged with the 11-step unified workflow (`docs/claude/current-process-workflow.md`). Comparative analysis decided: do not run the spec as a separate process — run it AS a project through the 11-step workflow.

---

## 0. What this produces

A per-project **knowledge library**: 10–16 reference skills in the TARGET project's `.claude/skills/`, capturing what the codebase IS (architecture decisions, failure history, config axes, domain theory) — complementing the plugin's process skills, which capture how to work.

| | Plugin process skills | Knowledge library (this pilot) |
|---|---|---|
| Answers | "HOW do we work?" | "WHAT is this codebase/domain?" |
| Examples | brainstorming, writing-plans | `<project>-failure-archaeology`, `<project>-architecture-contract` |
| Lives in | plugin `skills/` | target project `.claude/skills/` |
| Cadence | reused every task | built once, maintained via STEP 11 |

**Why combined with the workflow instead of standalone:** the workflow becomes the library's maintenance mechanism (STEP 11 lessons feed back into skills), and the library becomes the workflow's long-term memory (STEP 4 reads it instead of re-researching). Standalone, the library rots and the workflow re-learns everything per project.

---

## 1. Ground rules (non-negotiable)

1. **GROUND TRUTH ONLY.** Verify every command, flag, path, and claim against the target repo before stating it. Wrong runbooks are worse than none. `[VERIFIED/CITED/ASSUMED]` tags mandatory in all research output; nothing tagged `[ASSUMED]` may appear in a final skill stated as fact.
2. **Write fence.** Write ONLY inside `<target>/.claude/skills/` plus the workflow's own artifacts (`.planning/`, state files). The rest of the target repo is read-only. Git commits only per the target's `commit_docs`/`commit_atomic` config and user approval.
3. **Never auto-advance.** All 11-step human touchpoints apply. The original spec's "one question round then autopilot" model is explicitly overridden.
4. **Audience contract.** Every skill is written for a zero-context mid-level engineer or Sonnet-class model: imperative runbook voice, copy-pasteable commands, every jargon term defined once, tables and checklists over prose. Each skill states when NOT to use it and which sibling to use instead.
5. **No oversell.** Unproven things stay labeled `open` / `candidate`. Nothing may contradict the target project's CLAUDE.md or route around its change control.
6. **Volatile facts are date-stamped.** Every skill ends with a **"Provenance and maintenance"** section: one-line re-verification commands for anything that may drift (flag lists, version numbers, file paths).
7. **Embed knowledge.** Don't reference private/user-specific paths as load-bearing sources. If a fact matters, it lives in the skill body.
8. **Language.** Skills are authored in English (model-facing) unless the target project's own policy says otherwise.
9. **Description = triggers only.** Per `skills/writing-skills/SKILL.md` CSO rules: frontmatter `description` starts with "Use when...", lists triggering conditions/symptoms only, NEVER summarizes the skill's content or workflow.

---

## 2. Workflow mapping — how the campaign runs

| Workflow step | Campaign action | Checkpoint |
|---|---|---|
| **STEP 0** | Bootstrap in TARGET project (brownfield path). If state files exist, run `validate-state` first. Suggested config: `mode: interactive`, `parallelization: true`, `model_defaults` as in §4. | User confirms config |
| **STEP 1** | Fast Lane: automatic NO (multi-file, multi-wave campaign). | — |
| **STEP 2** | Brainstorm = the spec's question round. Ask AT MOST five questions, only what the repo cannot tell: (1) hardest live problem right now, (2) unwritten discipline rules no doc states, (3) who the audience is and what they do NOT know, (4) which past failures cost the most time, (5) what "beyond state of the art" means here. Fold answers into taxonomy adaptation. Output: REQUIREMENTS.md — one REQ-ID per skill. | User approves direction |
| **STEP 3** | Mode gate: score honestly. Expected outcome Mode B (multi-domain research + formal QA), but the score decides. | User approves mode |
| **STEP 4** | Research = the spec's Phase 1 discovery. Parallel agents over: README/manifest/contributor docs, build system, test suite + how it actually runs, CI config, docs dirs, **git history (what changed, what got reverted, what stalled on dead branches)**, TODO/FIXME hotspots, issue-shaped artifacts, generated-data/deploy conventions, project memory/notes. Output `.planning/research/*` with claim tags. This is the evidence base every author agent reads. | User reviews sufficiency |
| **STEP 5** | Spec: lock the **taxonomy** — final skill inventory (names, scopes, one-line content spec each), merge/split/cut decisions per §3, per-skill outline. | **User approves inventory** |
| **STEP 6** | Plan: one XML task per skill. `<model>` per §4, `<read_first>` = relevant research files + sibling inventory (names + descriptions, for cross-references), `<verify>` = grep-checkable (§5), waves per §4. Plan Checker runs if enabled. | User approves plan |
| **STEP 7** | Execute wave by wave. Fresh context per author agent. Zero file overlap by design (each skill = own directory) → parallel dispatch is safe. Escalation haiku→sonnet→opus if BLOCKED. | User approves each wave |
| **STEP 8** | UAT = **retrieval testing** (§6). This is the Iron-Law resolution — see below. | User drives |
| **STEP 9** | QA Gate = the spec's Phase 3: three reviewers (FACTUAL / DOCTRINE / USABILITY) + one fixer (§7). | User reviews findings |
| **STEP 10** | Skip (no release gate for a docs-only library) unless target policy requires it. | — |
| **STEP 11** | Ship: commit per config, SUMMARY.md **must embed the field notes (§8)**, STATE/ROADMAP updated. | User decides |

**Iron-Law resolution (why batch authoring is legal here):** `skills/writing-skills/SKILL.md` requires a failing test before any skill, but its own typology says **reference skills** are tested by *retrieval + gap testing*, not pressure scenarios. STEP 8 UAT (§6) IS that test, run per skill. Discipline-flavored skills (e.g. `<project>-change-control`) additionally get 1–2 pressure scenarios. The "STOP before next skill" rule is satisfied at wave granularity: no wave N+1 until wave N passes its verify checks.

---

## 3. Taxonomy — adapt, don't transcribe

Target 10–16 skills. Merge categories that are thin in this target, split ones that are deep, add domain categories the list doesn't imagine. The inventory is locked at STEP 5 based on STEP 4 evidence — never before.

### CORE (default: include, merge if thin)

| # | Skill | Content spec |
|---|---|---|
| 1 | `<project>-change-control` | How changes are classified, gated, reviewed HERE; non-negotiables with rationale AND the historical incident behind each |
| 2 | `<project>-debugging-playbook` | Symptom→triage table for this project's failure modes; traps that cost real time (each with its story); discriminating experiments |
| 3 | `<project>-failure-archaeology` | Chronicle: every major investigation, dead end, rejected fix, revert — as symptom → root cause → evidence → status. Mine git history hard |
| 4 | `<project>-architecture-contract` | Load-bearing design decisions and WHY; invariants that must hold; known-weak points stated plainly |
| 5 | `<domain>-reference` | Domain-theory pack a mid-level person lacks: the field's math/protocols/standards **as they apply HERE**, not a textbook |
| 6 | `<project>-config-and-flags` | Catalog of every config axis: options, defaults, production vs experimental, guards; how to add one (checklist); re-verification commands |
| 7 | `<project>-build-and-env` | Recreate the environment from scratch; known traps |
| 8 | `<project>-run-and-operate` | Running/deploying: command anatomy, data/artifact conventions, what output lands where |
| 9 | `<project>-diagnostics-and-tooling` | How to MEASURE instead of eyeball: diagnostic tools with interpretation guides; actual scripts in the skill's `scripts/` dir |
| 10 | `<project>-validation-and-qa` | What counts as evidence here; acceptance thresholds; certified/golden inventory; how to add tests |
| 11 | `<project>-docs-and-writing` | Docs of record, templates, house style |
| 12 | `<project>-external-positioning` | Papers/releases/ecosystem: novel vs known, what must be proven before claiming, reproducibility standards |

### ADVANCED (evidence-gated: include ONLY if STEP 4 found real ground truth)

| # | Skill | Include only if… |
|---|---|---|
| 13 | `<project>-<hardest-problem>-campaign` | STEP 2 Q1 named a concrete hard problem AND STEP 4 found enough evidence to write decision gates with EXPECTED numbers ("if you see X instead → branch to Y"), a ranked solution menu, fenced-off wrong paths, and a promotion protocol routed through change control. Success measurable, never judged by eye |
| 14 | `<project>-proof-and-analysis-toolkit` | The domain has first-principles analysis methods ("prove it, don't just install it") AND repo history contains at least one worked example per recipe |
| 15 | `<project>-research-frontier` | The project genuinely has open problems near state of the art. Each entry: why current SOTA fails, this project's specific asset, first three concrete steps IN THIS REPO, falsifiable "you have a result when…" milestone |
| 16 | `<project>-research-methodology` | The project runs hypothesis-driven experiments. Evidence bar: one mechanism must explain ALL observations including negatives and survive assigned adversarial refutation; hypotheses predict numbers before running; idea lifecycle from experiment flag to adopted change or documented retirement |

**Cut rule:** an ADVANCED skill with thin ground truth is not written thin — it is cut, and the gap is recorded in field notes. Inventing content to fill a template is the failure mode this protocol exists to prevent.

---

## 4. Wave structure and model assignment

| Wave | Skills | Model | Rationale |
|---|---|---|---|
| 1 — Evidence miners | failure-archaeology, config-and-flags, build-and-env, architecture-contract, domain-reference | `sonnet` | Mine research output + repo directly; no cross-skill deps |
| 2 — Builders | debugging-playbook, change-control, run-and-operate, diagnostics-and-tooling, validation-and-qa, docs-and-writing | `sonnet` | Cite Wave 1: playbook cites archaeology; change-control cites incident history |
| 3 — Synthesis | hardest-problem-campaign, proof-and-analysis-toolkit, research-frontier, research-methodology, external-positioning | `opus` | Judgment-heavy; read Waves 1–2 output |

Context budget rule applies: max 2–3 tasks per plan → each wave = 2–3 plans. Escalate, never compress.

---

## 5. Authoring contract (goes into every author agent's prompt)

- Format: `.claude/skills/<name>/SKILL.md`; YAML frontmatter with `name` (letters/numbers/hyphens) and trigger-rich `description` per §1.9.
- `<read_first>`: the relevant `.planning/research/*` files + the locked inventory list (sibling names + descriptions) so cross-references and "use sibling X instead" pointers are accurate.
- Every command/flag/path re-verified against the repo before inclusion (run it read-only or locate it in source).
- Scripts promised by the skill are shipped in the skill's `scripts/` dir and smoke-tested.
- Ends with "Provenance and maintenance": date + one-line re-verification command per volatile fact.
- `<verify>` (grep-checkable, <60s): file exists; frontmatter has `name` + `description` starting with "Use when"; body contains "When NOT to use" and "Provenance and maintenance"; no `[ASSUMED]` tags remain.

---

## 6. STEP 8 UAT — retrieval test protocol

For each skill (fresh-context subagent, library installed, no conversation history):

1. **Find test:** pose 2–3 real questions the skill should answer (drawn from its REQ-ID). Does the agent load the RIGHT skill from `description` alone? Miss = CSO/trigger gap.
2. **Answer test:** does the loaded skill actually answer correctly and completely? Miss = content gap.
3. **Command spot-check:** run 3 random commands from each runbook-type skill read-only; they must work as written.
4. **Negative test:** one question the skill should NOT answer — does it correctly point to the sibling?
5. Discipline-flavored skills (change-control): 1–2 pressure scenarios ("urgent hotfix, can we skip the gate?").

Failures → gap list → fix plans → back to STEP 7. UAT results recorded in `.planning/{phase}-UAT.md`.

---

## 7. STEP 9 QA — three reviewers + fixer

| Reviewer | Model | Checks | Severity guide |
|---|---|---|---|
| FACTUAL | `sonnet` | Re-verify flags/paths/commands/citations against repo; flag anything invented or stale | "Would it send an engineer down a wrong path?" → Critical |
| DOCTRINE | `opus` | Contradictions with target's rules or between skills; overstated claims; anything that routes around change control; unproven stated as fact | Routing around gates → Critical |
| USABILITY | `sonnet` | Trigger quality of descriptions; duplication (one home per fact, cross-refs elsewhere); self-containedness; scannability | Duplication → Important |

Fixer applies Critical + Important findings; Suggestions logged. Standard fix loop back to STEP 7 if structural.

---

## 8. Field notes — the pilot's second product

Maintain `.planning/pilot-FIELD-NOTES.md` throughout (not retrospectively). Record:

- Taxonomy misfits: which categories merged/split/cut and why; domain categories invented.
- Wave structure: did the dependency order hold? What actually blocked?
- Prompt rework: which author prompts failed first try, and the fix.
- UAT catch rate: find-misses vs answer-misses vs command failures — which test caught what.
- Reviewer overlap: did FACTUAL/DOCTRINE/USABILITY duplicate findings? Could two merge?
- Actuals: context budget per author, wall-clock per wave, escalations triggered.
- Anything the formalized skill must encode, forbid, or warn about.

**A pilot with a perfect library but empty field notes has failed** — the notes are what formalization is built from.

---

## 9. After the pilot — formalization deliverables (back in this plugin repo)

Only after field notes exist (writing-skills doctrine: baseline before skill):

| # | Deliverable | Notes |
|---|---|---|
| A | `skills/mining-project-knowledge/SKILL.md` | This protocol, corrected by field notes, as a reusable skill |
| B | `commands/mine-knowledge.md` | Slash-command entry point (pattern: `commands/init-project.md`) |
| C | `docs/claude/templates/knowledge-skill-template.md` | SKILL.md skeleton incl. Provenance section |
| D | Consumption hooks | `research-phase-guide.md`: STEP 4 agents load `<project>-*` skills before external research. `current-process-workflow.md`: STEP 7 failure recovery loads debugging-playbook before model escalation; STEP 9 reads validation-and-qa as evidence bar; STEP 1 Fast Lane auto-ineligible when task touches change-control-flagged areas; STEP 11 knowledge-sync sub-step (lessons → archaeology/playbook/config skills) |
| E | CLAUDE.md section + CHANGELOG | Version bump v5.9.0 or v6.0.0 |
| F (optional v2) | `validate-state` extension | WARN on stale "Provenance and maintenance" dates in knowledge skills |

---

## 10. Kickoff (paste in a session opened in the TARGET project workspace)

```
Run the knowledge-mining pilot per e:\firstwordflow\docs\specs\2026-07-07-knowledge-mining-pilot-protocol.md.
Start at STEP 0 (brownfield bootstrap). Treat the protocol's ground rules (§1) as binding.
Target: this repository. Maintain .planning/pilot-FIELD-NOTES.md from the first step.
```

(Or copy this file into the target repo first if cross-repo path references are undesirable.)
