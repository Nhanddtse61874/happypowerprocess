---
phase: init-project
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - docs/claude/config-schema.md
  - docs/claude/templates/settings.json
  - docs/claude/templates/gitignore
autonomous: true
requirements: [INIT-01, INIT-02, INIT-03]

must_haves:
  truths:
    - "config-schema.md có đủ 9 fields với 6 cột (Field/Type/Default/Valid values/Description/Mid-workflow impact)"
    - "settings.json template valid JSON với Option C allowlist"
    - "gitignore template có .worktrees/ + .claude/settings.local.json + commented .planning/"
  artifacts:
    - "docs/claude/config-schema.md"
    - "docs/claude/templates/settings.json"
    - "docs/claude/templates/gitignore"
  key_links:
    - "settings.json template parses as valid JSON"
    - "config-schema.md defaults match templates/config.json defaults"
---

<!--
  WAVE 1 — Foundation files. Tất cả task độc lập, run in parallel.
  Phụ thuộc: NONE (Wave 1)
  Spec reference: docs/specs/2026-05-06-init-project-skill-design.md
-->

<wave number="1" description="Foundation templates + schema doc — independent, parallel">

  <task type="auto">
    <name>Create config-schema.md (single source of truth for config fields)</name>

    <model>haiku</model>

    <read_first>
      docs/claude/templates/config.json
      docs/claude/current-process-workflow.md
      docs/specs/2026-05-06-init-project-skill-design.md
    </read_first>

    <files>
      <create>docs/claude/config-schema.md</create>
    </files>

    <action>
      Create `docs/claude/config-schema.md` containing the 9-field schema table per spec Section "Components > 1. config-schema.md".

      Structure:
      1. Title: `# Config Schema (happypowerprocess)`
      2. Intro paragraph: explain this is single source of truth, used by /init-project + /update-config skills
      3. Markdown table with 6 columns: Field | Type | Default | Valid values | Description | Mid-workflow impact
      4. 10 rows (note: commit_docs has 2 sub-fields, model_defaults has nested object — list as written in spec, can use dotted notation like `commit_docs.state_files`)
      5. Section "## How to Use This Schema" — 3 sub-sections:
         - For `/init-project` skill: read defaults, apply to `.planning/config.json`
         - For `/update-config` skill: validate input against valid values, show mid-workflow impact warnings
         - Schema verification: skill checks defaults match `templates/config.json`

      AVOID:
      - Don't add fields not in spec (no scope creep)
      - Don't change default values (must match templates/config.json)
      - Don't use HTML, only Markdown

      Corresponds to REQ-ID: INIT-01 (foundation schema for both skills).
    </action>

    <verify>
      <automated>powershell -Command "if ((Get-Content e:/firstwordflow/docs/claude/config-schema.md -Raw) -match 'mode' -and (Get-Content e:/firstwordflow/docs/claude/config-schema.md -Raw) -match 'commit_atomic' -and (Get-Content e:/firstwordflow/docs/claude/config-schema.md -Raw) -match 'workflow.plan_check') { Write-Output 'PASS' } else { Write-Output 'FAIL'; exit 1 }"</automated>
      <!-- Verifies file exists and contains 3 sample field names from schema. <60s. -->
    </verify>

    <done>
      File `docs/claude/config-schema.md` exists. Contains markdown table with all 9 fields: mode, granularity, parallelization, commit_docs.state_files, commit_docs.planning_artifacts, commit_atomic, model_profile, model_defaults, workflow.research, workflow.plan_check. Defaults column matches values in `templates/config.json`. "How to Use This Schema" section present.
    </done>

    <stack>plugin-meta</stack>
  </task>

  <task type="auto">
    <name>Create settings.json template (Option C workflow-aware permission allowlist)</name>

    <model>haiku</model>

    <read_first>
      .claude/settings.json
      docs/specs/2026-05-06-init-project-skill-design.md
    </read_first>

    <files>
      <create>docs/claude/templates/settings.json</create>
    </files>

    <action>
      Create `docs/claude/templates/settings.json` with EXACT content per spec Section "Components > 4. settings.json":

      ```json
      {
        "permissions": {
          "allow": [
            "Read",
            "Glob",
            "Grep",
            "Bash(git status:*)",
            "Bash(git log:*)",
            "Bash(git diff:*)",
            "Bash(git branch:*)",
            "Bash(ls:*)",
            "Bash(cat:*)",
            "Bash(pwd)"
          ]
        }
      }
      ```

      AVOID:
      - Don't add Edit, Write, or destructive Bash commands (Option C deliberately excludes these)
      - Don't add stack-specific commands (Option D rejected)
      - Don't add comments inside JSON (invalid syntax)

      Corresponds to REQ-ID: INIT-02 (permission allowlist defaults).
    </action>

    <verify>
      <automated>powershell -Command "try { Get-Content e:/firstwordflow/docs/claude/templates/settings.json -Raw | ConvertFrom-Json | Out-Null; Write-Output 'PASS' } catch { Write-Output 'FAIL: invalid JSON'; exit 1 }"</automated>
      <!-- Verifies file exists and is valid JSON. <60s. -->
    </verify>

    <done>
      File `docs/claude/templates/settings.json` exists. Parses as valid JSON. Contains `permissions.allow` array with exactly 10 entries: Read, Glob, Grep, Bash(git status:*), Bash(git log:*), Bash(git diff:*), Bash(git branch:*), Bash(ls:*), Bash(cat:*), Bash(pwd). No Edit/Write/destructive commands present.
    </done>

    <stack>plugin-meta</stack>
  </task>

  <task type="auto">
    <name>Create gitignore template</name>

    <model>haiku</model>

    <read_first>
      .gitignore
      docs/specs/2026-05-06-init-project-skill-design.md
    </read_first>

    <files>
      <create>docs/claude/templates/gitignore</create>
    </files>

    <action>
      Create `docs/claude/templates/gitignore` with EXACT content per spec Section "Components > 5. gitignore":

      ```
      # happypowerprocess workflow
      .worktrees/
      .claude/settings.local.json

      # Conditional (added when commit_docs.planning_artifacts: false):
      # .planning/
      ```

      File name: `gitignore` (NO leading dot — this is template, init-project skill will rename/append to `.gitignore` in target project).

      AVOID:
      - Don't include `.git/` (already gitignored by git itself)
      - Don't include node_modules/ or other stack-specific entries (out of scope)
      - Don't uncomment `.planning/` line (conditional, controlled by skill at runtime)

      Corresponds to REQ-ID: INIT-03 (gitignore base entries).
    </action>

    <verify>
      <automated>powershell -Command "$content = Get-Content e:/firstwordflow/docs/claude/templates/gitignore -Raw; if ($content -match '\.worktrees/' -and $content -match '\.claude/settings\.local\.json' -and $content -match '# \.planning/') { Write-Output 'PASS' } else { Write-Output 'FAIL'; exit 1 }"</automated>
      <!-- Verifies all 3 required entries present. <60s. -->
    </verify>

    <done>
      File `docs/claude/templates/gitignore` exists. Contains `.worktrees/`, `.claude/settings.local.json`, and commented `# .planning/`. No leading dot in filename.
    </done>

    <stack>plugin-meta</stack>
  </task>

</wave>

<!--
  PLAN CHECKER quick pass (mental review):
  1. Requirement Coverage — INIT-01/02/03 covered. ✓
  2. Task Completeness — model + read_first + action + verify + done. ✓
  3. Dependency Correctness — Wave 1, no deps, acyclic. ✓
  4. Key Links Planned — schema defaults match config.json (artifact wiring). ✓
  5. Scope Sanity — 3 tasks/plan. ✓
  6. Verification Derivation — must_haves trace to phase goal. ✓
  7. Context Compliance — N/A (no CONTEXT.md). ✓
  8. Nyquist Compliance — automated verify <60s each. ✓
  9. Cross-Plan Data Contracts — schema doc consumed by Plan 2. ✓
  10. CLAUDE.md Compliance — XML format per workflow. ✓
  11. Research Resolution — N/A (research skipped). ✓
-->
