---
name: mining-project-knowledge
description: Use when onboarding a complex or brownfield project and you need a persistent, per-project knowledge library — architecture decisions, failure history, config axes, domain theory, debugging playbooks — captured as `.claude/skills/<project>-*` reference skills. Runs as STEP 0.5 (Knowledge Bootstrap). Triggered by /mine-knowledge, "mine project knowledge", "build a knowledge library", or a new/inherited codebase whose tribal knowledge lives only in people's heads.
---

# Mining Project Knowledge

> **Status: candidate.** This skill has not yet been pressure/retrieval-tested per `writing-skills`. Its first real run (the pilot on a real project) IS its first test — capture field notes (§ Field notes) and harden the skill afterward. Do not present its outputs as proven until validated.

## Overview

Produces a **per-project knowledge library**: 10–16 *reference* skills written into the target project's `.claude/skills/`, capturing **what the codebase IS** — complementing the plugin's *process* skills, which capture **how to work**.

| | Plugin process skills | Knowledge library (this skill) |
|---|---|---|
| Answers | "HOW do we work?" | "WHAT is this codebase/domain?" |
| Examples | `brainstorming`, `writing-plans` | `<project>-failure-archaeology`, `<project>-architecture-contract` |
| Lives in | plugin `skills/` | target project `.claude/skills/` |
| Cadence | reused every task | built once (here), maintained by STEP 11 |

**Why it belongs in the workflow, not standalone:** the workflow becomes the library's maintenance mechanism (STEP 11 lessons feed back in) and the library becomes the workflow's long-term memory (STEP 4 reads it instead of re-researching). Standalone, the library rots and the workflow re-learns everything per project.

Full design + rationale: `docs/specs/2026-07-07-knowledge-mining-pilot-protocol.md`.

## When to use

- Onboarding a mature/brownfield codebase whose knowledge is undocumented (in people's heads, scattered in git history, or nowhere).
- A project pushing hard problems where re-fighting settled battles is expensive.
- User runs `/mine-knowledge`.

## When NOT to use — pick the sibling instead

- **A single feature/bugfix** → the normal 11-step workflow. This skill builds a *reference library*, not shipped code.
- **A trivial or greenfield project with no tribal knowledge yet** → skip; there is no ground truth to mine.
- **You just need to know one fact about the codebase** → read the code / ask; don't run a campaign.
- **Authoring a general, cross-project technique skill** → `writing-skills` (this skill only produces project-specific reference skills that live in the *project's* `.claude/skills/`, never in plugin core).

## Ground rules (non-negotiable)

1. **GROUND TRUTH ONLY.** Verify every command, flag, path, and claim against the target repo before stating it. Tag research `[VERIFIED/CITED/ASSUMED]`; nothing `[ASSUMED]` may appear in a final skill stated as fact. Wrong runbooks are worse than none.
2. **Write fence.** Write ONLY inside `<target>/.claude/skills/` plus the workflow's own artifacts (`.planning/`, state files). Rest of the target repo is read-only.
3. **Never auto-advance.** All 11-step human touchpoints apply.
4. **Audience:** zero-context mid-level engineer or Sonnet-class model. Imperative runbook voice, copy-pasteable commands, jargon defined once, tables/checklists. Each skill states when NOT to use it and which sibling to use.
5. **No oversell.** Unproven things stay labeled `open`/`candidate`. Never contradict the target's CLAUDE.md or route around its change control.
6. **Date-stamp volatile facts.** Every generated skill ends with a "Provenance and maintenance" section: one-line re-verification command per fact that can drift.
7. **Description = triggers only** (per `writing-skills` CSO): "Use when…", no workflow summary.

## How it runs — as a project through the 11-step workflow

This skill does not invent a new process; it runs the campaign THROUGH the existing workflow. Summary (full mapping in the protocol §2):

| Workflow step | Campaign action |
|---|---|
| STEP 0 | Bootstrap in target (brownfield). `validate-state` if state exists. |
| STEP 2 | Ask ≤5 questions the repo can't answer: hardest live problem · unwritten discipline rules · audience + what they don't know · costliest past failures · what "beyond state of the art" means here. |
| STEP 4 | Discovery: parallel agents over README/manifests, build, tests+how-run, CI, docs, **git history (reverts, dead branches)**, TODO/FIXME, issues, deploy conventions, project memory. Claim tags mandatory. |
| STEP 5 | Lock the taxonomy (below) — names + one-line scope each; merge/split/cut per evidence. **[user approves inventory]** |
| STEP 6 | One XML task per skill; `<model>` per §Waves; `<verify>` grep-checkable (§5 of protocol). |
| STEP 7 | Author wave by wave. One skill = own directory → zero file overlap → parallel-safe. |
| STEP 8 | UAT = **retrieval testing** (find / answer / command-spot-check / negative test per skill). This is the Iron-Law-satisfying test for reference skills. |
| STEP 9 | QA = three reviewers (FACTUAL / DOCTRINE / USABILITY) + fixer. |
| STEP 11 | Ship; SUMMARY embeds field notes. |

## Taxonomy — adapt, don't transcribe (target 10–16)

Lock at STEP 5 from STEP 4 evidence. Merge thin categories, split deep ones, add domain categories the list doesn't imagine.

**CORE (default include, merge if thin):**
`<project>-change-control` · `<project>-debugging-playbook` · `<project>-failure-archaeology` · `<project>-architecture-contract` · `<domain>-reference` · `<project>-config-and-flags` · `<project>-build-and-env` · `<project>-run-and-operate` · `<project>-diagnostics-and-tooling` · `<project>-validation-and-qa` · `<project>-docs-and-writing` · `<project>-external-positioning`

**ADVANCED (include ONLY if STEP 4 found real ground truth — else cut, don't pad):**
`<project>-<hardest-problem>-campaign` · `<project>-proof-and-analysis-toolkit` · `<project>-research-frontier` · `<project>-research-methodology`

**Cut rule:** an ADVANCED skill with thin ground truth is **cut**, not written thin. Inventing content to fill a template is the exact failure this skill exists to prevent — record the gap in field notes instead.

Per-skill content specs: protocol §3.

### Architecture: relationship to init-project §d2 (scope split)

`init-project` §d2 (Architecture Detection) writes a **layout label** (Clean Architecture / N-Tier / …) into `.claude/stack-skills/<stack>/SKILL.md` `## Architecture` — an *execution-time hint* for implementers (STEP 7). `<project>-architecture-contract` captures **load-bearing decisions, invariants, and known-weak points** — for research/planning (STEP 4/5). Different consumers, different depth: **no single fact lives in both.** When both exist, `architecture-contract` cross-references the §d2 layout label, and the §d2 section links to `architecture-contract` for the "why".

## Waves & model assignment

| Wave | Skills | Model |
|---|---|---|
| 1 — Evidence miners | failure-archaeology, config-and-flags, build-and-env, architecture-contract, domain-reference | `sonnet` |
| 2 — Builders | debugging-playbook, change-control, run-and-operate, diagnostics-and-tooling, validation-and-qa, docs-and-writing | `sonnet` |
| 3 — Synthesis | hardest-problem-campaign, proof-and-analysis-toolkit, research-frontier, research-methodology, external-positioning | `opus` |

Wave 2 cites Wave 1 (playbook cites archaeology; change-control cites incident history). Max 2-3 skills per plan (context budget).

## UAT — retrieval test (STEP 8), per skill

Fresh-context subagent, library installed, no history:
1. **Find:** 2-3 real questions the skill should answer — does the agent load the RIGHT skill from `description` alone? Miss = trigger/CSO gap.
2. **Answer:** does it answer correctly + completely? Miss = content gap.
3. **Command spot-check:** run 3 random commands from each runbook read-only; must work as written.
4. **Negative:** one question it should NOT answer → does it point to the sibling?
5. Discipline-flavored skills (`change-control`): 1-2 pressure scenarios.

Failures → gap list → fix plans → back to STEP 7.

## QA — three reviewers + fixer (STEP 9)

| Reviewer | Model | Checks |
|---|---|---|
| FACTUAL | `sonnet` | re-verify flags/paths/commands/citations vs repo; flag invented/stale — "would it send an engineer down a wrong path?" → Critical |
| DOCTRINE | `opus` | contradictions with target rules or between skills; overstated claims; anything routing around change control | 
| USABILITY | `sonnet` | trigger quality; duplication (one home per fact); self-containedness; scannability |

Fixer applies Critical + Important. Suggestions logged.

## How the library feeds the workflow (consumption + maintenance hooks)

Once `.claude/skills/<project>-*` exists, the 11-step workflow consumes and maintains it (each hook is a no-op until the library exists):

- **STEP 1 Fast Lane:** task touching an area `<project>-change-control` flags dangerous → not Fast-Lane-eligible.
- **STEP 4 Research:** load `<project>-*` FIRST; research only the gaps.
- **STEP 7 Execute:** on BLOCKED, consult `<project>-debugging-playbook` before escalating model.
- **STEP 9 QA:** use `<project>-validation-and-qa` as the evidence bar.
- **STEP 11 Ship (Knowledge Sync):** write lessons back — new bug → `failure-archaeology` + `debugging-playbook`; new config → `config-and-flags`; new decision → `architecture-contract`.

## Field notes — the second product

Maintain `.planning/pilot-FIELD-NOTES.md` throughout (not retrospectively): taxonomy misfits (merged/split/cut + why), wave-order reality, prompt rework, UAT catch rate, reviewer overlap, actuals (context/wall-clock/escalations), and anything the skill must encode/forbid next. **A perfect library with empty field notes has failed** — the notes are what hardens this candidate skill.

## Provenance and maintenance

- Created 2026-07-07. Formalizes `docs/specs/2026-07-07-knowledge-mining-pilot-protocol.md`.
- Status: **candidate** — validate on first real run, then harden per field notes.
- Re-verify taxonomy vs protocol: `grep -c '<project>-' docs/specs/2026-07-07-knowledge-mining-pilot-protocol.md`.
- Consumption hooks are mirrored in `CLAUDE.md` (STEP 1/4/7/9/11) and `docs/claude/current-process-workflow.md` — if you change a hook, change it in both.
