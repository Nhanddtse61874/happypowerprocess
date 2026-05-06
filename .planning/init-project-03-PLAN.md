---
phase: init-project
plan: 03
type: execute
wave: 3
depends_on: [init-project-02]
files_modified:
  - .claude-plugin/plugin.json
  - CLAUDE.md
autonomous: true
requirements: [INIT-06]

must_haves:
  truths:
    - "plugin.json keywords mention init-project + update-config (discoverability)"
    - "Plugin CLAUDE.md documents both new commands so plugin maintainers know they exist"
  artifacts:
    - ".claude-plugin/plugin.json (updated)"
    - "CLAUDE.md (updated)"
  key_links:
    - "plugin.json keywords drive marketplace search/discovery"
    - "Plugin CLAUDE.md is read by AI for context — must reference new skills"
---

<!--
  WAVE 3 — Plugin integration. Run after Wave 2 (skills exist).
  Phụ thuộc: Wave 2 (skills/init-project/SKILL.md, skills/update-config/SKILL.md)
  Spec reference: docs/specs/2026-05-06-init-project-skill-design.md
-->

<wave number="3" description="Plugin integration — register new skills and document">

  <task type="auto">
    <name>Update .claude-plugin/plugin.json with new keywords for discoverability</name>

    <model>haiku</model>

    <read_first>
      .claude-plugin/plugin.json
      .cursor-plugin/plugin.json
    </read_first>

    <files>
      <modify>.claude-plugin/plugin.json</modify>
      <modify>.cursor-plugin/plugin.json</modify>
    </files>

    <action>
      Update `.claude-plugin/plugin.json` and `.cursor-plugin/plugin.json` to add new keywords reflecting init-project and update-config skills.

      Read current keywords array. Add these entries at the END of array (preserve existing order):
      - "init-project"
      - "bootstrap"
      - "config-management"
      - "project-setup"

      Do NOT:
      - Change version (that's separate task)
      - Modify name/description/author/repository fields
      - Remove existing keywords
      - Reorder existing keywords

      Apply same change to BOTH manifest files (.claude-plugin and .cursor-plugin) to keep them in sync (current state shows both manifests are mirrored).

      Corresponds to REQ-ID: INIT-06 (plugin manifest registration).
    </action>

    <verify>
      <automated>powershell -Command "$claude = Get-Content e:/firstwordflow/.claude-plugin/plugin.json -Raw | ConvertFrom-Json; $cursor = Get-Content e:/firstwordflow/.cursor-plugin/plugin.json -Raw | ConvertFrom-Json; if ($claude.keywords -contains 'init-project' -and $claude.keywords -contains 'bootstrap' -and $cursor.keywords -contains 'init-project') { Write-Output 'PASS' } else { Write-Output 'FAIL'; exit 1 }"</automated>
      <!-- Verifies both manifests have new keywords. <60s. -->
    </verify>

    <done>
      Both `.claude-plugin/plugin.json` and `.cursor-plugin/plugin.json` parse as valid JSON. `keywords` array contains: init-project, bootstrap, config-management, project-setup (in addition to existing entries). All other fields unchanged.
    </done>

    <stack>plugin-meta</stack>
  </task>

  <task type="auto">
    <name>Update plugin CLAUDE.md to document /init-project + /update-config commands</name>

    <model>sonnet</model>

    <read_first>
      CLAUDE.md
      skills/init-project/SKILL.md
      skills/update-config/SKILL.md
      docs/specs/2026-05-06-init-project-skill-design.md
    </read_first>

    <files>
      <modify>CLAUDE.md</modify>
    </files>

    <action>
      Update plugin's `CLAUDE.md` (root level) to document the 2 new slash commands.

      Insert a NEW section after "## Personalized Plugin Mode (This Workspace)" section, BEFORE "## Code-Level Behavioral Guidelines" section.

      New section content:

      ```markdown
      ## Bootstrap Commands (New in v5.4.0)

      Two slash commands handle project setup and config management:

      - **`/init-project`**: Bootstrap state files, config, CLAUDE.md, permissions, gitignore. Skill: `skills/init-project/SKILL.md`. Asks 2 prompts (project name, primary stack), supports `--force` flag for re-init with backup. Brownfield auto-detection via Glob markers.
      - **`/update-config`**: Interactive config update with field validation and mid-workflow impact warnings. Skill: `skills/update-config/SKILL.md`. Reads from `docs/claude/config-schema.md` (single source of truth for all 9 config fields).

      Both skills follow the 11-step workflow: `/init-project` produces STEP 0 artifacts, `/update-config` is invoked by user trigger phrases at any time.

      Schema reference: `docs/claude/config-schema.md`
      ```

      Do NOT:
      - Modify any existing section content
      - Change "## BRANCHING RULE" or "## General" sections
      - Remove the contributor guidelines (they remain mandatory for upstream PRs per existing text)
      - Touch the GitNexus block at the end (auto-managed)

      Length target: ~12 lines added.

      Corresponds to REQ-ID: INIT-06 (documentation of new commands in plugin CLAUDE.md).
    </action>

    <verify>
      <automated>powershell -Command "$content = Get-Content e:/firstwordflow/CLAUDE.md -Raw; if ($content -match '/init-project' -and $content -match '/update-config' -and $content -match 'config-schema\.md' -and $content -match '## BRANCHING RULE') { Write-Output 'PASS' } else { Write-Output 'FAIL'; exit 1 }"</automated>
      <!-- Verifies new commands documented AND existing BRANCHING RULE preserved. <60s. -->
    </verify>

    <done>
      Plugin `CLAUDE.md` contains new "Bootstrap Commands" section documenting `/init-project` and `/update-config`. Section references `docs/claude/config-schema.md`. All pre-existing sections (Personalized Plugin Mode, Code-Level Guidelines, AI Agent guidelines, PR Requirements, BRANCHING RULE, General, Design UX/UI, GitNexus block) preserved unchanged.
    </done>

    <stack>plugin-meta</stack>
  </task>

</wave>

<!--
  PLAN CHECKER quick pass:
  1. Requirement Coverage — INIT-06 covered. ✓
  2. Task Completeness — model + read_first + action + verify + done. ✓
  3. Dependency Correctness — Wave 3 depends on Wave 2, acyclic. ✓
  4. Key Links Planned — plugin.json discoverability + CLAUDE.md documentation. ✓
  5. Scope Sanity — 2 tasks/plan. ✓
  6. Verification Derivation — must_haves trace to discoverability + documentation. ✓
  7. Context Compliance — N/A. ✓
  8. Nyquist Compliance — automated verify <60s. ✓
  9. Cross-Plan Data Contracts — depends on skills from Plan 2. ✓
  10. CLAUDE.md Compliance — XML format, surgical changes (preserve existing). ✓
  11. Research Resolution — N/A. ✓
-->
