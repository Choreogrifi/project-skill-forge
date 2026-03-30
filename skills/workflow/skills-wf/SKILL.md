---
name: skills-wf
description: Author new skills, detect skill gaps in the active session, and audit skill content quality. State transitions are delegated to the skillforge CLI.
metadata:
  skill-type: workflow
  version: "1.0"
  memory-file: workflow/skills.md
  disable-model-invocation: true
---

# Skills

Authors new skills, monitors sessions for skill gaps, and audits content quality.
State is managed by the `skillforge` CLI — never by this skill directly.

## CLI Delegation

For all state operations, emit the appropriate command and stop:

| Intent | Command |
|---|---|
| List all skills | `skillforge ls` |
| Activate a skill | `skillforge activate <name>` |
| Put skill in review | `skillforge review <name>` |
| Deactivate a skill | `skillforge deactivate <name>` |
| Decommission a skill | `skillforge rm <name>` |
| Fix violations | `skillforge audit` |
| Environment check | `skillforge doctor` |

## Subflows

Load the relevant subflow only when the user's intent matches:

| File | Load when |
|---|---|
| `subflows/skills-create-sf.md` | User wants to create or scaffold a new skill |
| `subflows/skills-detect-sf.md` | Monitoring session for skill gaps (background protocol) |
| `subflows/skills-audit-sf.md` | Auditing skill content, frontmatter, or memory references |
| `subflows/skills-propose-sf.md` | User wants to propose a new skill to the Skill Forge repository |
| `subflows/skills-refine-sf.md` | User wants to improve an existing skill's content |

## Guidelines

- Always write new skills to `${SKILLFORGE_DIR}/skills/` — never to the working directory.
- Never delete skill directories — decommission via `skillforge rm <name>`.
- After authoring a skill, always end with: `Run: skillforge activate <name>`
- Keep `SKILL.md` lean — move content >10 lines into `references/` or subflow files.
- Memory scaffolding (stub creation, list, archive) is owned by `memory-wf`.

## References

- `references/marketplace-overlap.md` — plugin conflict data; check when creating skills that overlap with Claude plugins
- `references/mcp-playbook.md` — MCP server setup (generic); see `references/mcp-playbook-claude.md` for Claude-specific config
