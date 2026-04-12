# Research Phase Guide (Step 4)

Research phase runs after Mode Selection Gate and before Spec Writing. Purpose: gather evidence before planning, instead of planning from assumptions.

## When to Run

- After Step 3 (Mode Gate) — both Mode A and Mode B run research
- Skip if: Fast Lane eligible, or `workflow.research: false` in config.json

## Critical Startup Protocol (each research agent)

Before researching anything, agent must:
1. Load CONTEXT.md if it exists — locked decisions constrain scope
2. Load `.planning/config.json` — read validation settings
3. Read CLAUDE.md — project directives override all recommendations

## Mode A: Research (Solo) — 2 agents

| Agent | Task | Output section |
|---|---|---|
| Stack Researcher | Patterns, conventions, best practices for current stack | Stack Findings |
| Pitfall Researcher | Known issues, anti-patterns, gotchas with the approach under consideration | Pitfalls & Anti-patterns |

## Mode B: Research (Team) — 4 agents + Synthesizer

| Agent | Task | Output file |
|---|---|---|
| Stack Researcher | Current stack with versions and rationale | `.planning/research/STACK.md` |
| Feature Researcher | Table stakes vs differentiators vs anti-features | `.planning/research/FEATURES.md` |
| Architecture Researcher | Component boundaries, data flow, build order | `.planning/research/ARCHITECTURE.md` |
| Pitfall Researcher | Common mistakes, prevention strategies, edge cases | `.planning/research/PITFALLS.md` |
| **Research Synthesizer** | Consolidate 4 outputs into actionable summary | `.planning/research/SUMMARY.md` |

**Synthesizer runs after all 4 agents** — SUMMARY.md is the primary input for Spec Writing (Step 5).

## Claim Provenance (CRITICAL)

Every factual claim must be tagged with its source:

| Tag | Meaning | Trust level |
|---|---|---|
| `[VERIFIED: npm registry]` | Tool-confirmed | HIGH |
| `[CITED: docs.example.com]` | Official documentation | HIGH-MEDIUM |
| `[ASSUMED]` | Training knowledge, needs user validation | LOW |

**Never present assumed knowledge as verified fact** — especially for compliance, security, performance claims.

**Honest research philosophy:**
- Verify before asserting — training data may be 6-18 months stale
- Report gaps honestly — "couldn't find X" is more valuable than fabricating
- Flag LOW confidence clearly — surfaces items needing validation
- Treat evidence as driver, not confirmation bias

## Tool Priority for Research

1. **Context7** — Library APIs, configs, versions (HIGH trust)
2. **Official docs / WebFetch** — READMEs, changelogs (HIGH-MEDIUM)
3. **WebSearch** — Ecosystem patterns (needs cross-verification)

## Research Workflow (each agent)

| Step | Action |
|------|--------|
| 1 | Load phase context, CONTEXT.md, project constraints |
| 2 | Identify research domains |
| 2.5 | *Rename/refactor phases only:* Runtime state inventory (stored data, live config, OS registration, secrets, build artifacts) |
| 2.6 | *External dependency phases:* Environment availability audit via bash probes |
| 3 | Execute research per tool priority |
| 4 | Map requirements to tests, identify gaps (if enabled) |
| 5 | Quality check: domains covered, negatives verified, confidence assigned |
| 6 | Write output file |

## RESEARCH.md Structure (Mode A output)

```markdown
# {Phase} Research

**Date:** {YYYY-MM-DD}
**Mode:** A
**Input:** Brainstorm output + approved design direction

<user_constraints>
## User Constraints (from CONTEXT.md, if exists)
### Locked Decisions
### Claude's Discretion
### Deferred Ideas (OUT OF SCOPE)
</user_constraints>

<phase_requirements>
## Phase Requirements
[Map REQ-IDs to research findings]
</phase_requirements>

## Standard Stack

{Stack findings with claim tags}

## Architecture Patterns

{Pattern findings}

## Don't Hand-Roll

{Libraries/tools that already exist — do not reimplement}

## Common Pitfalls

{Anti-patterns to avoid}

## Assumptions Log

{All ASSUMED claims — need user validation}

## Open Questions

{Items not found or needing clarification}

## Sources

{URLs, docs referenced}
```

## Human Touchpoint

After research completes: User reviews `.planning/{phase}-RESEARCH.md` (Mode A) or `.planning/research/SUMMARY.md` (Mode B), confirms sufficient information before continuing to Step 5.

If gaps exist: add a targeted research agent for the specific area — do not restart the entire phase.

## Dispatcher Routing

| Task type | Owner |
|---|---|
| `research-phase-mode-a` | Main session (2 parallel subagents) |
| `research-phase-mode-b` | `team-orchestrator` (4 parallel subagents + synthesizer) |