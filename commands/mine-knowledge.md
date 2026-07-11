---
description: "Build a per-project knowledge library (.claude/skills/<project>-*) — STEP 0.5 Knowledge Bootstrap"
---

Invoke the **happypowerprocess:mining-project-knowledge** skill to build a knowledge library for this project.

The skill runs a decision-gated campaign THROUGH the 11-step workflow (STEP 0.5 — Knowledge Bootstrap) to capture, as `.claude/skills/<project>-*` reference skills, what this codebase IS: architecture decisions + invariants, failure history, config axes, build/run/diagnostics, validation discipline, and (where real ground truth exists) domain theory and the hardest-problem campaign.

It handles:
- Discovery over git history, build, tests, CI, docs, TODO/FIXME, deploy conventions (claim-tagged `[VERIFIED/CITED/ASSUMED]`)
- ≤5 questions the repo cannot answer (hardest problem, unwritten rules, audience, costliest failures)
- Taxonomy lock (10–16 skills; cut ADVANCED skills that lack ground truth — never pad)
- Wave-based authoring, retrieval-test UAT, three-reviewer QA
- Field notes (`.planning/pilot-FIELD-NOTES.md`)

**Ground truth only** — every command/flag/path verified against the repo; nothing `[ASSUMED]` stated as fact. Writes only inside `.claude/skills/` + workflow artifacts; rest of the repo is read-only.

**Status:** the skill is a *candidate* — its first run is also its first real test. Expect to harden it from the field notes afterward.

Pass the user's full invocation message to the skill so it can detect the target and any modifiers.
