---
skill: skills-wf
skill-type: workflow
description: Context for authoring skills, detecting gaps, and auditing skill content
last-updated: 2026-03-30
---

## Workflow Context

- Skill state transitions (activate, deactivate, decommission) always delegated to `skillforge` CLI
- Never delete skill directories — rename suffix to retire, or use `skillforge rm <name>`
- After authoring, always emit: `Run: skillforge activate <name>`
- Skills root: `${SKILLFORGE_DIR}/skills/<type>/<name>/`
- Deferred skill plans: `${SKILLFORGE_DIR}/memory/deferred-skills/`

## Naming Convention

- All skill directory names end with a type suffix: `-sme` or `-wf`
- Name field in frontmatter must match the directory name exactly
- Validated by pre-commit hook and `skillforge audit`

## Skill Types

| Type | Suffix | Purpose |
|---|---|---|
| `sme-persona` | `-sme` | Domain expertise and standards |
| `workflow` | `-wf` | Step-by-step automation and routing |

## Key Patterns

- SKILL.md body ≤ 30 lines — move overflow to `references/` subdir
- Subflow files have no frontmatter — plain markdown loaded lazily
- Memory stub creation: delegate to `memory-wf`
- Marketplace overlap check: read `references/marketplace-overlap.md` before creating
