---
phase: process-2.0
plan: 01
type: execute
wave: 1
depends_on: []
files_modified: [docs/claude/config-schema.md, docs/claude/templates/config.json, docs/claude/memory-store-guide.md]
autonomous: false
requirements: [CFG-01, MEM-02]

must_haves:
  truths:
    - "A developer can set workflow.memory_recall / workflow.telemetry / workflow.sandbox_verify in config, each defaulting off"
    - "A developer reading memory-store-guide.md knows exactly where process-memory lives, its file schema, and its boundary vs the knowledge library"
  artifacts:
    - "docs/claude/config-schema.md"
    - "docs/claude/templates/config.json"
    - "docs/claude/memory-store-guide.md"
  key_links:
    - "config-schema.md flag rows match templates/config.json keys exactly (no schema drift)"
    - "memory-store-guide.md states process-memory NEVER writes into .claude/skills/<project>-*"
---

<!-- Source: docs/specs/2026-07-11-process-2.0-memory-observability-design.md (D1, D2, D6, D7) -->

<wave number="1" description="Foundation — disjoint files, parallel-safe">

  <task type="auto">
    <name>Add Process 2.0 config flags (CFG-01)</name>
    <model>opus</model>
    <read_first>
      docs/claude/config-schema.md
      docs/claude/templates/config.json
      docs/specs/2026-07-11-process-2.0-memory-observability-design.md
    </read_first>
    <files>
      <modify>docs/claude/config-schema.md</modify>
      <modify>docs/claude/templates/config.json</modify>
    </files>
    <action>
      REQ CFG-01. Add three boolean flags to the existing `workflow.*` group, each default `false`:
      `workflow.memory_recall` (MEM-01/02/03/04), `workflow.telemetry` (OBS-01/02),
      `workflow.sandbox_verify` (VER-01/02).
      - In docs/claude/templates/config.json: add the three keys inside the existing `"workflow"` object,
        all `false`.
      - In docs/claude/config-schema.md: add one schema-table row per flag (columns:
        Field | Type | Default | Valid values | Description | Mid-workflow impact), using dotted names
        `workflow.memory_recall` etc., Type `bool`, Default `false`. Add a Display-Grouping entry per the
        file's "Rule for new fields" (join the existing workflow group or own index — do NOT hardcode index range).
      MUST edit BOTH files — a Default mismatch triggers the hard-abort "Plugin schema drift. Reinstall plugin."
      Do NOT add memory/telemetry/verify logic here — flags only. Do NOT touch any other config field.
    </action>
    <verify>
      <automated>grep -c "memory_recall\|telemetry\|sandbox_verify" docs/claude/templates/config.json</automated>
    </verify>
    <done>
      grep above returns 3; config-schema.md contains a row for each of the three `workflow.*` flags with
      Default `false`; no other config field changed.
    </done>
  </task>

  <task type="auto">
    <name>Author memory-store-guide.md (MEM-02 schema + boundary)</name>
    <model>opus</model>
    <read_first>
      docs/specs/2026-07-11-process-2.0-memory-observability-design.md
      skills/mining-project-knowledge/SKILL.md
      docs/claude/state-files-guide.md
    </read_first>
    <files>
      <create>docs/claude/memory-store-guide.md</create>
    </files>
    <action>
      REQ MEM-02 (schema) + design D1/D2/D6. Write the process-memory store guide (English), covering:
      - Location: `<project>/.claude/memory/`, committed to git (portable), NOT under `.planning/`.
      - Format: a `MEMORY.md` index (one line per lesson) + one `.md` file per lesson.
      - Per-lesson frontmatter: `scope: global|project`, `type`, `tags`, `date`.
      - Boundary rule: process-memory captures "how we worked"; the knowledge library
        (`.claude/skills/<project>-*`) captures "what the code is". Process-memory MUST NOT write into
        `.claude/skills/<project>-*`. Both are no-op when absent.
      - Recall-on-demand convention: grep/glob top-K (tag + keyword + recency), never load the whole index.
      - English normalization: lessons are normalized to English on write (plugin English-only policy).
      Reference the design spec as the source. Do NOT implement recall/capture logic here — this is the schema/contract doc.
    </action>
    <verify>
      <automated>grep -l "scope: global" docs/claude/memory-store-guide.md && grep -c "\.claude/skills/<project>-\*" docs/claude/memory-store-guide.md</automated>
    </verify>
    <done>
      File exists; documents location `.claude/memory/`, the `scope/type/tags/date` frontmatter, the
      "never write into .claude/skills/<project>-*" boundary, recall-on-demand, and English normalization.
    </done>
  </task>

</wave>
