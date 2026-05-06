---
phase: init-project
plan: 02
type: execute
wave: 2
depends_on: [init-project-01]
files_modified:
  - skills/init-project/SKILL.md
  - skills/update-config/SKILL.md
autonomous: true
requirements: [INIT-04, INIT-05]

must_haves:
  truths:
    - "User can run /init-project → 2 prompts (project name, stack) → all state files created"
    - "User can run /init-project on initialized project → 4-choice menu shows"
    - "User can run /update-config → see indexed config table → pick fields → update with validation"
    - "init-project choice [2] chains to /update-config"
  artifacts:
    - "skills/init-project/SKILL.md"
    - "skills/update-config/SKILL.md"
  key_links:
    - "Both skills reference docs/claude/config-schema.md (created in Wave 1)"
    - "init-project copies all templates from docs/claude/templates/ (existing + Wave 1 new)"
    - "update-config writes to .planning/config.json with mid-workflow impact warnings"
---

<!--
  WAVE 2 — Main skill files. Run after Wave 1 (foundation files exist).
  Phụ thuộc: Wave 1 (config-schema.md, settings.json, gitignore templates)
  Spec reference: docs/specs/2026-05-06-init-project-skill-design.md
-->

<wave number="2" description="Main skill files — depend on Wave 1 templates">

  <task type="auto">
    <name>Create skills/init-project/SKILL.md (bootstrap project skill)</name>

    <model>sonnet</model>

    <read_first>
      skills/version/SKILL.md
      docs/claude/templates/CLAUDE.md
      docs/claude/templates/PROJECT.md
      docs/claude/templates/STATE.md
      docs/claude/templates/config.json
      docs/claude/templates/settings.json
      docs/claude/templates/gitignore
      docs/claude/config-schema.md
      docs/specs/2026-05-06-init-project-skill-design.md
    </read_first>

    <files>
      <create>skills/init-project/SKILL.md</create>
    </files>

    <action>
      Create `skills/init-project/SKILL.md` implementing the init-project skill per spec Section "Components > 2. init-project SKILL.md" + spec Section "Data Flow > Init flow".

      Required structure:

      1. **Frontmatter** (YAML):
         ```yaml
         ---
         name: init-project
         description: Bootstrap project state files, config, CLAUDE.md, permissions, and gitignore for happypowerprocess workflow. Use when user types /init-project or wants to start a new project.
         ---
         ```

      2. **Body sections** (in order):

         a. **Plugin path detection** — instruct AI to detect platform via env vars:
            - Windows: `$env:USERPROFILE\.claude\plugins\marketplaces\happypowerprocess\`
            - macOS/Linux: `$HOME/.claude/plugins/marketplaces/happypowerprocess/`
            - Detection logic: check `$env:OS == "Windows_NT"` or `[[ "$OSTYPE" == "msys" ]]`

         b. **Detection phase** — check existence of:
            - `.planning/STATE.md` (idempotency trigger)
            - `.git/` (git init prompt trigger)
            - Brownfield markers via Glob: `*.csproj`, `*.sln`, `package.json`, `Cargo.toml`, `pom.xml`, `pubspec.yaml`

         c. **Idempotency gate** (if STATE.md exists) — present 4-choice menu (per spec Section "Data Flow > Init flow Already initialized"):
            - [1] Abort (default)
            - [2] Update config only → invoke `/update-config` skill, exit
            - [3] Re-init missing files only (skip existing)
            - [4] Force re-init (BACKUP existing → overwrite)
            - Support `--force` flag: skip menu, jump to choice 4 with 1 confirmation

         d. **Brownfield handling** — if markers found:
            - Display: "📦 Brownfield detected: {detected stacks}"
            - Skip stack prompt (auto-fill)
            - Add note to STATE.md: "Brownfield project. Run /brainstorm to map architecture."

         e. **Interactive prompts** (greenfield):
            - Q1: "Project name?" → validate regex `[a-zA-Z0-9_\- ]+`, max 3 retries
            - Q2: "Primary stack? (dotnet/react/react-native/angular/iot/mixed)" → validate against valid list, max 3 retries

         f. **File creation** (in order, idempotent for choice [3]):
            - mkdir `.planning/`, `.planning/research/`, `docs/specs/`
            - Copy templates from `{plugin-path}/docs/claude/templates/` to project:
              - `PROJECT.md` → `.planning/PROJECT.md` (fill `{PROJECT_NAME}`, `{STACK}`)
              - `REQUIREMENTS.md` → `.planning/REQUIREMENTS.md`
              - `ROADMAP.md` → `.planning/ROADMAP.md`
              - `STATE.md` → `.planning/STATE.md` (fill `Phase: STEP 0`, `Next: Run /brainstorm for first feature`)
              - `config.json` → `.planning/config.json`
              - `CLAUDE.md` → `CLAUDE.md` (fill `{PROJECT_NAME}`, `{INIT_DATE}`)
              - `settings.json` → `.claude/settings.json`
              - Create `.claude/settings.local.json` with empty allow array
            - Append `gitignore` template entries to project's `.gitignore` (idempotent — skip if entry exists, create new file if `.gitignore` không tồn tại)

         g. **Git init prompt** (if no `.git`):
            - Display: "ℹ Folder chưa có git repo."
            - Prompt: "Init git? [y/n]"
            - y → run `git init`
            - n → skip with note

         h. **Commit prompt**:
            - Display list of files to commit
            - Prompt: "Auto-commit 'init: project state files'? [y/n]"
            - y → `git add` per `commit_docs` config + `git commit -m "init: project state files"`
            - n → skip (files staged or not based on user preference)

         i. **STATE.md audit entry**:
            - Append to "Notes" section: `- {YYYY-MM-DD HH:MM}: Project initialized via /init-project (Mode: greenfield/brownfield, Stack: {STACK})`

         j. **Output summary**:
            - List files created
            - Git status (initialized? committed?)
            - Next-step guidance: "Run /brainstorm to start your first feature"

      3. **Error handling** (per spec Section "Error Handling > Init-project errors"):
         - Plugin path missing → abort with clear message
         - Permission denied → abort + roll back partial files (track created list)
         - User Ctrl+C → cleanup partial files
         - Invalid input → re-prompt (max 3 retries) → abort
         - Brownfield ambiguous (monorepo) → display detected, ask user confirm

      AVOID:
      - Don't run deep questioning about vision (defer to STEP 2 brainstorming)
      - Don't analyze code architecture (defer to brainstorm)
      - Don't auto-init git silently (always ask)
      - Don't auto-commit silently (always ask)
      - Don't overwrite existing files in choice [3] (create only missing)

      Length target: ~120 lines (excluding frontmatter).

      Corresponds to REQ-ID: INIT-04 (init-project skill).
    </action>

    <verify>
      <automated>powershell -Command "$content = Get-Content e:/firstwordflow/skills/init-project/SKILL.md -Raw; if ($content -match 'name: init-project' -and $content -match 'idempotency' -and $content -match 'brownfield' -and $content -match '--force') { Write-Output 'PASS' } else { Write-Output 'FAIL'; exit 1 }"</automated>
      <!-- Verifies frontmatter + key sections present. <60s. -->
    </verify>

    <done>
      File `skills/init-project/SKILL.md` exists. Frontmatter has `name: init-project` + description. Body covers all 10 sections (a-j) per spec. Error handling section present. References `config-schema.md` for validation. References templates in `docs/claude/templates/`. Mentions `--force` flag.
    </done>

    <stack>plugin-meta</stack>
  </task>

  <task type="auto">
    <name>Create skills/update-config/SKILL.md (interactive config update skill)</name>

    <model>sonnet</model>

    <read_first>
      skills/version/SKILL.md
      docs/claude/config-schema.md
      docs/claude/templates/config.json
      docs/specs/2026-05-06-init-project-skill-design.md
    </read_first>

    <files>
      <create>skills/update-config/SKILL.md</create>
    </files>

    <action>
      Create `skills/update-config/SKILL.md` implementing the update-config skill per spec Section "Components > 3. update-config SKILL.md" + spec Section "Data Flow > Update-config flow".

      Required structure:

      1. **Frontmatter** (YAML):
         ```yaml
         ---
         name: update-config
         description: Display current .planning/config.json and let user update specific fields with validation and mid-workflow impact warnings. Use when user types /update-config or says "update config", "change config", "change model", "change commit strategy".
         ---
         ```

      2. **Body sections** (in order):

         a. **Pre-check** — verify `.planning/config.json` exists:
            - NO → abort: "Config not found. Run /init-project first."
            - YES → continue

         b. **Plugin path detection** — same as init-project (Windows/macOS-Linux env vars)

         c. **Load schema + current config**:
            - Read `{plugin-path}/docs/claude/config-schema.md` (AI reads markdown table mentally for valid values + impact)
            - Read `.planning/config.json` (parse JSON)
            - If schema doc missing → abort: "Plugin schema doc malformed. Reinstall plugin."
            - If config malformed JSON → abort with line error, suggest `/init-project --force`

         d. **Display current config** — formatted table with 10 numbered entries (commit_docs split into 2 lines, model_defaults split into 3):
            ```
            Current config:
              1. mode:                              interactive
              2. granularity:                       standard
              3. parallelization:                   true
              4. commit_docs.state_files:           true
              5. commit_docs.planning_artifacts:    true
              6. commit_atomic:                     true
              7. model_profile:                     balanced
              8. model_defaults.mechanical:         haiku
                 model_defaults.standard:           sonnet
                 model_defaults.complex:            opus
              9. workflow.research:                 true
             10. workflow.plan_check:               true
            ```

         e. **User picks fields**:
            - Prompt: "Which fields to change? (số, comma-separated, hoặc 'all')"
            - Validate input: must be comma-separated indices in range 1-10, or 'all'
            - Invalid → re-prompt (max 3 retries)
            - Empty input → exit cleanly (no changes)

         f. **Per-field interactive update loop**:
            - For each picked field:
              - Display: "[field-name] Current: {value}. Valid: {options from schema}"
              - Prompt: "New value?"
              - Validate against schema valid values (max 3 retries)
              - Store in pending changes dict
              - If max retries exceeded → skip current field, continue to next

         g. **Summary diff**:
            ```
            Summary:
              granularity:    standard → coarse
              commit_atomic:  true → false
            ```
            - Prompt: "Apply? [y/n]"
            - n → exit, no changes

         h. **Write config**:
            - Update `.planning/config.json` with new values
            - Detect concurrent edit (mtime check before write)

         i. **Side-effects**:
            - If `commit_docs.planning_artifacts` changed:
              - true → false: append `.planning/` to `.gitignore` (idempotent)
              - false → true: remove `.planning/` line from `.gitignore` if present

         j. **Mid-workflow impact warnings** (per field changed, from schema):
            - Read schema "Mid-workflow impact" column
            - Display: "⚠ {field} changed: {impact}"
            - Examples:
              - granularity → "Affects unplanned phases only. Already-planned phases unchanged."
              - commit_atomic → "Applies from next task. Existing commits unchanged."

         k. **STATE.md audit entry**:
            - Append to "Notes" section: `- {YYYY-MM-DD HH:MM}: Config updated via /update-config (changed: {field-list})`

         l. **Output summary**:
            - "✓ Updated .planning/config.json"
            - List warnings
            - List side-effects (if any)

      3. **Error handling** (per spec Section "Error Handling > Update-config errors"):
         - Config not found → abort with hint
         - Malformed JSON → abort with error
         - Invalid index → re-prompt
         - Invalid value → re-prompt max 3, then skip field
         - Schema missing → abort
         - Write fails → abort, original config unchanged
         - .gitignore update fails → warn but don't abort
         - Concurrent edit → abort

      AVOID:
      - Don't open external editor (Option C rejected)
      - Don't auto-update without user confirm
      - Don't validate field values against custom rules — only against schema valid values
      - Don't modify other state files (only .planning/config.json + .gitignore conditional)

      Length target: ~70 lines (excluding frontmatter).

      Corresponds to REQ-ID: INIT-05 (update-config skill).
    </action>

    <verify>
      <automated>powershell -Command "$content = Get-Content e:/firstwordflow/skills/update-config/SKILL.md -Raw; if ($content -match 'name: update-config' -and $content -match 'config-schema' -and $content -match 'mid-workflow' -and $content -match 'planning_artifacts') { Write-Output 'PASS' } else { Write-Output 'FAIL'; exit 1 }"</automated>
      <!-- Verifies frontmatter + key sections present. <60s. -->
    </verify>

    <done>
      File `skills/update-config/SKILL.md` exists. Frontmatter has `name: update-config` + description với trigger phrases. Body covers all 12 sections (a-l). References `config-schema.md`. Side-effects section handles `.gitignore` toggle. Mid-workflow impact warnings present. Error handling per spec.
    </done>

    <stack>plugin-meta</stack>
  </task>

</wave>

<!--
  PLAN CHECKER quick pass:
  1. Requirement Coverage — INIT-04/05 covered. ✓
  2. Task Completeness — model + read_first + action + verify + done. ✓
  3. Dependency Correctness — Wave 2 depends on Wave 1, acyclic. ✓
  4. Key Links Planned — both skills wire to schema doc. ✓
  5. Scope Sanity — 2 tasks/plan. ✓
  6. Verification Derivation — must_haves trace to user-observable behavior. ✓
  7. Context Compliance — N/A. ✓
  8. Nyquist Compliance — automated verify <60s. ✓
  9. Cross-Plan Data Contracts — schema doc from Plan 1 consumed here. ✓
  10. CLAUDE.md Compliance — XML format. ✓
  11. Research Resolution — research skipped, but pattern from /version skill known. ✓
-->
