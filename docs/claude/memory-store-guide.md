# Process-Memory Store Guide

Process-memory is the workflow's long-term memory of **how we worked** — model escalations, plan misfires, config choices, and workflow gotchas. It lives in the repo, is readable by both AI and humans, and persists across context resets and machines.

> This is the **schema / contract** doc for the process-memory store. It defines *where memory lives, its file format, and its boundary* — not the recall or capture logic. Source of truth: `docs/specs/2026-07-11-process-2.0-memory-observability-design.md` (D1, D2, D6). Recall (MEM-01/04) is a STEP-0 agent step; capture/write-back is a STEP-11 step that feeds recall and has no separate v1 REQ-ID; both are documented in the workflow, not here.

## Location

```
{project-root}/
└── .claude/
    └── memory/
        ├── MEMORY.md        — index, one line per lesson
        ├── <lesson-1>.md    — one file per lesson
        ├── <lesson-2>.md
        └── ...
```

- **Path:** `<project>/.claude/memory/`.
- **Committed to git → portable.** A `git clone` restores the store on a new machine; no re-init needed. This is why the store lives in the repo, not in a machine-local cache.
- **NOT under `.planning/`.** `.planning/` can be gitignored by `commit_docs.planning_artifacts: false`. If process-memory lived there, that flag would silently un-commit it and portability would break. `.claude/` is already inside the write-fence and is always committed, so the store can never be silently dropped.
- **Distinct from native/global memory.** User prefs and cross-project feedback live in the Claude Code native store (`~/.claude/projects/<slug>/memory/`), which is out of scope for this guide. Process-memory is per-project and shared with the team via git.

## File format

Mirrors the native memory convention: a small always-scannable index plus one file per lesson.

### `MEMORY.md` — the index

One line per lesson, linking to its file. Kept small so it can be listed cheaply. Example:

```markdown
# Process Memory — Index

- [haiku under-plans multi-file waves](haiku-underplans-multifile-waves.md) — escalate to sonnet for >2 files
- [config: commit_atomic=false batches per wave](config-commit-atomic-batching.md) — set before first wave
- [plan-check loop stalls without a phase map](plancheck-needs-phase-map.md)
```

### One `.md` file per lesson

Each lesson is its own file with YAML frontmatter, then the lesson body.

| Frontmatter field | Type | Values | Meaning |
|---|---|---|---|
| `scope` | enum | `global` \| `project` | `scope: global` = user/feedback/reference-flavored, few and stable; `scope: project` = specific to this codebase's workflow. |
| `type` | string | e.g. `escalation`, `plan-misfire`, `config`, `gotcha` | The kind of lesson; used to rank recall. |
| `tags` | list | free-form keywords | Retrieval keys (stack, step, symptom). |
| `date` | date | `YYYY-MM-DD` (absolute) | When the lesson was captured; feeds recency ranking. |

Example lesson file:

```markdown
---
scope: project
type: escalation
tags: [haiku, multi-file, wave, model-defaults]
date: 2026-07-11
---

# haiku under-plans multi-file waves

When a wave touches more than two files, haiku consistently missed cross-file
wiring and the task came back BLOCKED. Escalating to sonnet on the first retry
resolved it. Consider bumping `model_defaults` for waves with >2 files.
```

- `scope: global` lessons are the rare, stable ones; keeping them tagged lets recall surface them across projects while the many `scope: project` lessons are retrieved on demand.
- Use **absolute dates** (`2026-07-11`), never relative ("today") — matching the state-files convention.

## Boundary rule (critical)

There are two on-disk stores and exactly **one home per fact**:

| Store | Path | Captures | Curated? |
|---|---|---|---|
| **Process-memory** (this doc) | `.claude/memory/` | **how we worked** — process/workflow lessons | No (auto, English-normalized) |
| **Knowledge library** | `.claude/skills/<project>-*` | **what the code is** — architecture, failure archaeology, config axes | **Yes** — ground-truth, human-approved |

**Tie-break:** a lesson about the *workflow/process* → process-memory; a fact about the *codebase* → the knowledge library (`.claude/skills/<project>-*`).

**Process-memory MUST NOT write into `.claude/skills/<project>-*`.** The knowledge library is ground-truth-only and human-gated (see `skills/mining-project-knowledge/SKILL.md`); process-memory is auto-captured and un-curated. Mixing an auto store into a curated one is a doctrine clash — so the write-fence is hard: process-memory writes only inside `.claude/memory/`, never into `.claude/skills/<project>-*`.

**Both are no-op when absent.** If `.claude/memory/` does not exist, recall/capture do nothing. If `.claude/skills/<project>-*` does not exist, the knowledge-sync hooks do nothing. Neither store is required for the workflow to run; each just enriches it when present. This matches the existing conditional-hook convention.

## Recall-on-demand

Recall (STEP 0, MEM-04) **never loads the whole `MEMORY.md` into context every session.** As the store grows, a preloaded index would inflate per-session context without bound. Instead:

1. Run a **Grep/Glob query** over `.claude/memory/` for the current task's keywords.
2. **Rank** hits by `tag match + keyword overlap + recency`.
3. **Load only the top-K** matching lesson files.

Per-session context cost stays flat as the store grows. v1 uses lexical (grep) recall; a semantic vector index (MEM-05) is a deferred v2 upgrade. The `SessionStart` hook may inject a one-line reminder at startup, but the STEP 0 agent recall step is the primary, reliable path (it also covers mid-session resume, where `SessionStart` does not fire).

## English normalization

Lessons are **normalized to English on write.** Even when the session was conducted in another language, the STEP 11 capture write-back translates the lesson content to English before persisting it. This satisfies the plugin English-only source policy — every file in `.claude/memory/` is English, regardless of chat language (D6).

## Principles

- One home per fact — process-memory for "how we worked", knowledge library for "what the code is".
- Committed to git, inside `.claude/`, never under `.planning/` — so resume and portability always work.
- Index stays small; lessons are retrieved on demand, not preloaded.
- Flag-for-removal maintenance (MEM-03) is human-approved — process-memory never auto-deletes.
- All lesson content is English; use absolute dates.

## Provenance and maintenance

- Created 2026-07-11 from `docs/specs/2026-07-11-process-2.0-memory-observability-design.md` (D1/D2/D6), REQ MEM-02.
- Re-verify the boundary rule is stated: `grep -c "\.claude/skills/<project>-\*" docs/claude/memory-store-guide.md`.
- Re-verify frontmatter contract: `grep -n "scope: global" docs/claude/memory-store-guide.md`.
