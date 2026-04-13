# Version Control & README Improvement — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Sync all version references to 5.3.0, rewrite README.md with comprehensive 17-section structure, backfill CHANGELOG.md, and add a `/version` skill.

**Architecture:** Update version in `package.json` (source of truth) first, then sync to `marketplace.json`. Update release docs (RELEASE-NOTES, CHANGELOG). Rewrite README from spec. Create version skill as new `skills/version/SKILL.md`.

**Tech Stack:** Markdown, JSON, shields.io static badges

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `package.json` | Modify | Update version to `5.3.0` |
| `marketplace.json` | Modify | Update version to `5.3.0` |
| `RELEASE-NOTES.md` | Modify | Prepend v5.3.0 section |
| `CHANGELOG.md` | Modify | Backfill v5.1.0, v5.2.0, add v5.3.0 |
| `README.md` | Rewrite | Full 17-section rewrite |
| `skills/version/SKILL.md` | Create | New `/version` skill |

---

## Task 1: Update version in package.json and marketplace.json

**Files:**
- Modify: `package.json`
- Modify: `marketplace.json`

- [ ] **Step 1: Update package.json version**

Change `version` field from `"5.0.7"` to `"5.3.0"`:

```json
{
  "name": "superpowers",
  "version": "5.3.0",
  "type": "module",
  "main": ".opencode/plugins/superpowers.js"
}
```

- [ ] **Step 2: Update marketplace.json version**

Change `plugins[0].version` from `"5.2.0"` to `"5.3.0"`:

```json
{
  "name": "happypowerprocess",
  "description": "Personalized AI workflow plugin: Superpowers + GSD + AI Team orchestration",
  "owner": {
    "name": "Nhan"
  },
  "plugins": [
    {
      "name": "happypowerprocess",
      "description": "Unified 11-step AI workflow: Superpowers execution skills + GSD state persistence + optional AI Team orchestration.",
      "version": "5.3.0",
      "source": {
        "source": "github",
        "repo": "Nhanddtse61874/happypowerprocess"
      },
      "author": {
        "name": "Nhan"
      }
    }
  ]
}
```

- [ ] **Step 3: Verify both files**

Run:
```bash
grep '"version"' package.json marketplace.json
```
Expected: both show `"5.3.0"`

- [ ] **Step 4: Commit**

```bash
git add package.json marketplace.json
git commit -m "chore: bump version to 5.3.0 in package.json and marketplace.json"
```

---

## Task 2: Update RELEASE-NOTES.md

**Files:**
- Modify: `RELEASE-NOTES.md`

- [ ] **Step 1: Prepend v5.3.0 section**

Add the following content at the top of `RELEASE-NOTES.md`, after the `# Release Notes` heading and before the existing `## v5.2.0` section:

```markdown
## v5.3.0 (2026-04-13)

### Per-Task Model Config & Granular Commit Control

- **Per-task model assignment** — each plan task now has a `<model>` element (`haiku`/`sonnet`/`opus`). Planner assigns from `model_defaults` in config. User can override before execution.
- **Granular commit control** — `commit_docs` changed from boolean to `{state_files: bool, planning_artifacts: bool}`. `commit_atomic` controls per-task vs per-wave commits.
- **Config management** — new "update config" anytime mechanism. Resume sessions display config with keep/edit prompt. Mid-workflow config changes warn about impact on completed steps.
- **Pre-task confirmation** — Step 7 now shows model + files before each task, asks if user wants to switch model. Commit strategy confirmed before first wave.
- **Cleaned up config** — removed redundant fields (verifier, nyquist_validation, auto_advance). All workflow docs translated to English.

### Version Control & README

- **Single source of truth** — `package.json` is the canonical version. All references (`marketplace.json`, badges, RELEASE-NOTES, CHANGELOG) sync from it.
- **README rewrite** — 17-section structure with TOC, badges, project structure, configuration guide, prerequisites, troubleshooting/FAQ, contributing link, and attribution.
- **CHANGELOG backfill** — added entries for v5.1.0 and v5.2.0 from RELEASE-NOTES.

### Version Skill

- **`/version` command** — new skill that reads `package.json` and displays the current plugin version. Quick way to verify which version is installed.

---
```

- [ ] **Step 2: Verify structure**

Run:
```bash
head -5 RELEASE-NOTES.md
```
Expected: heading line, blank, `## v5.3.0 (2026-04-13)`

- [ ] **Step 3: Commit**

```bash
git add RELEASE-NOTES.md
git commit -m "docs: add v5.3.0 release notes"
```

---

## Task 3: Backfill CHANGELOG.md

**Files:**
- Modify: `CHANGELOG.md`

- [ ] **Step 1: Add v5.1.0, v5.2.0, and v5.3.0 entries**

Prepend the following above the existing `## [5.0.5]` entry (after the `# Changelog` heading):

```markdown
## [5.3.0] - 2026-04-13

### Added

- **Version skill** — `/version` command reads `package.json` and displays current plugin version
- **README rewrite** — 17-section structure with TOC, badges, project structure, configuration guide, prerequisites, troubleshooting, contributing, attribution
- **Per-task model assignment** — `<model>` element in plan XML (`haiku`/`sonnet`/`opus`) with `model_defaults` in config
- **Granular commit control** — `commit_docs` object (`state_files` + `planning_artifacts`), `commit_atomic` (per task vs per wave)
- **Config management** — "update config" anytime, resume session config display with keep/edit prompt
- **Pre-task confirmation** — Step 7 shows model + files before each task

### Changed

- **Version source of truth** — `package.json` is now the canonical version reference
- **CHANGELOG backfill** — added v5.1.0 and v5.2.0 entries from RELEASE-NOTES

### Fixed

- Removed redundant config fields (`verifier`, `nyquist_validation`, `auto_advance`)

## [5.2.0] - 2026-04-08

### Added

- **11-step unified workflow** — GSD state persistence + research phase + wave-based execution + UAT gate combined with Superpowers Mode A/B + QA gate + DevOps phase
- **State files** — PROJECT.md, STATE.md, REQUIREMENTS.md, ROADMAP.md, `.planning/config.json` persist across sessions
- **Research phase** — Mode A (2 agents) / Mode B (4 agents + Synthesizer) with [VERIFIED/CITED/ASSUMED] claim provenance
- **Goal-backward planning** — Observable Truths → Required Artifacts → Required Wiring → must_haves frontmatter
- **Nyquist Rule** — automated verify steps must complete in <60s
- **Plan Checker** — 11-dimension validation before execution
- **Wave execution** — intra-wave file overlap check, fresh context per task, atomic commits, worktree isolation
- **9 human touchpoints** — AI never auto-advances without user confirmation
- **marketplace.json** — plugin marketplace install support

### Changed

- **Implementer skills rewritten** — React/TS (TanStack Query v6, Zustand, RHF+Zod), Angular/TS (Signals, RxJS takeUntilDestroyed, OnPush), IoT Edge (MQTT QoS, LWT, BLE GATT, TLS/mTLS), React Native/TS (typed navigation, MMKV, CodePush)
- **All 12 agents aligned** with unified workflow (REQ-ID format, goal-backward methodology, STATE.md updates)
- **Plugin renamed** to `happypowerprocess`
- **All platform configs updated** — Claude Code, Cursor, Codex, OpenCode, Gemini

## [5.1.0] - 2026-04-06

### Added

- **Initial personalized fork** from obra/superpowers
- **AI Team orchestration agents** — phase leads, orchestrator, dispatcher
- **GSD workflow documentation** stubs
- **CLAUDE.md** with workspace-specific execution rules

```

- [ ] **Step 2: Verify entry count**

Run:
```bash
grep "^## \[" CHANGELOG.md
```
Expected: `[5.3.0]`, `[5.2.0]`, `[5.1.0]`, `[5.0.5]` — four entries

- [ ] **Step 3: Commit**

```bash
git add CHANGELOG.md
git commit -m "docs: backfill CHANGELOG with v5.1.0, v5.2.0, and add v5.3.0"
```

---

## Task 4: Create version skill

**Files:**
- Create: `skills/version/SKILL.md`

- [ ] **Step 1: Create skills/version/ directory and SKILL.md**

Create `skills/version/SKILL.md` with the following content:

```markdown
---
name: version
description: Display the current plugin version. Use when user asks what version is installed or types /version.
---

Read the file `package.json` in the plugin root directory. Extract the `version` field.

Display the result in this exact format:

```
Superpowers Plugin — v{version}
Repository: https://github.com/Nhanddtse61874/happypowerprocess
```

Do not modify any files. This is a read-only operation.
```

- [ ] **Step 2: Verify file exists**

Run:
```bash
cat skills/version/SKILL.md
```
Expected: file content with frontmatter (`name: version`, `description: ...`) and instructions.

- [ ] **Step 3: Commit**

```bash
git add skills/version/SKILL.md
git commit -m "feat: add /version skill to display current plugin version"
```

---

## Task 5: Rewrite README.md

**Files:**
- Rewrite: `README.md`

This is the largest task. The full README content follows the 17-section structure from the spec.

- [ ] **Step 1: Write full README.md**

Replace the entire content of `README.md` with all 17 sections as specified in the design doc (`docs/superpowers/specs/2026-04-13-version-control-readme-design.md`). Key sections:

1. **Header + badges** — title, one-liner tagline, shields.io badges (version 5.3.0, MIT license, platform support)
2. **Table of Contents** — anchor links to all sections
3. **What Is This?** — 4 short paragraphs (what, problem, solution, origin)
4. **How It Works** — intro sentence + 11-step table (keep existing table content)
5. **Runtime Modes** — Mode A summary + Mode B summary + links to docs
6. **Skills Library** — 3 tables (Process/Execution/Quality) with Description + Activates When columns
7. **Implementer Skills** — keep existing table with intro sentence
8. **Project Structure** — tree view with 1-line descriptions
9. **State Files** — tree view + explanation table (5 rows)
10. **Planning Quality Gates** — 3 subsections (Research, Plan, Execution), 3-5 lines each
11. **Prerequisites** — Git, Node.js >= 18, coding agent
12. **Installation** — all platforms (Claude Code marketplace/local, Codex, OpenCode, Cursor) + verify + update
13. **Configuration** — `.planning/config.json` table (6 keys) + sample JSON + how to update
14. **Troubleshooting / FAQ** — 4-row problem/solution table
15. **Contributing** — links to CLAUDE.md and PR template, 3 key rules
16. **Attribution** — credit obra/superpowers
17. **License** — MIT + link to LICENSE

Writing principles:
- Short, clear sentences. No filler.
- Every feature described with what it does AND how to use it.
- Badges show `5.3.0`.

- [ ] **Step 2: Verify section count**

Run:
```bash
grep "^## " README.md | wc -l
```
Expected: 17 (one per section, excluding the `#` title)

- [ ] **Step 3: Verify badges show correct version**

Run:
```bash
grep "5.3.0" README.md
```
Expected: at least 1 match (version badge)

- [ ] **Step 4: Verify TOC anchor links**

Run:
```bash
grep "^- \[" README.md | head -20
```
Expected: TOC entries with `(#section-name)` anchor links

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: rewrite README with 17-section structure, badges, and comprehensive docs"
```

---

## Task 6: Final verification

**Files:**
- Read-only verification across all modified files

- [ ] **Step 1: Verify version consistency**

Run:
```bash
grep -h '"version"' package.json marketplace.json
grep "5.3.0" README.md RELEASE-NOTES.md CHANGELOG.md
```
Expected: all files reference `5.3.0`

- [ ] **Step 2: Verify all files are committed**

Run:
```bash
git status
```
Expected: clean working tree

- [ ] **Step 3: Verify skill file exists**

Run:
```bash
ls skills/version/SKILL.md
```
Expected: file exists

- [ ] **Step 4: Review git log**

Run:
```bash
git log --oneline -6
```
Expected: 5 new commits (version bump, release notes, changelog, version skill, readme rewrite)
