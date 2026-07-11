---
description: "Scan the process-memory store and flag duplicate/stale/superseded/contradictory lessons for human approval — never auto-deletes (MEM-03)"
---

Invoke the **happypowerprocess:memory-maintenance** skill to run a cleanup pass over this project's process-memory store (`.claude/memory/`).

The skill scans the store (per `docs/claude/memory-store-guide.md`) and **flags** entries that no longer earn their place — duplicate, stale, superseded, or contradictory lessons — then presents them for approval. It **does not auto-delete** (pillar #1, never auto-advance): the human approves every removal, one candidate at a time.

It handles:
- Locating the store — no-op with a report if `.claude/memory/` is absent
- Sorting candidates into four buckets (duplicate / stale / superseded / contradictory), each with cited evidence
- Presenting a grouped report where nothing is changed yet
- Applying only the removals/edits the human approves, and updating `MEMORY.md` accordingly

**Flag, don't delete** — every deletion is human-approved. **Write fence** — the skill touches only `.claude/memory/`, never the knowledge library (`.claude/skills/<project>-*`). The controller commits; the skill does not `git commit`.

**Two entry points:** this is the on-demand one. The same skill also runs an inline flag-and-report pass at STEP 11 Ship (that wiring lives in the workflow docs).

Pass the user's full invocation message to the skill so it can detect any modifiers.
