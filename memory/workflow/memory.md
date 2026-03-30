---
skill: memory-wf
skill-type: workflow
description: Context for memory write rules, routing table, and stub scaffolding
last-updated: 2026-03-30
---

## Workflow Context

- Always reads the target file before proposing a change
- All writes require explicit user approval — never write without showing a diff
- Never deletes memory content — archive with `<!-- archived: YYYY-MM-DD: <reason> -->` prefix
- Stub scaffolding (create), listing (audit), and system toggle subflows available

## Memory Routing Table

| User Intent | Target File |
|---|---|
| SME domain knowledge or standard | `${SKILLFORGE_DIR}/memory/sme/<skill-name>.md` |
| Workflow-specific context or convention | `${SKILLFORGE_DIR}/memory/workflow/<skill-name>.md` |

## Constraints

- Maximum 40 lines per memory file body — propose splitting if exceeded
- If target file does not exist: propose creating it with correct frontmatter schema
- Update `last-updated` field on every write
- `memory/shared/identity.md` and `memory/shared/workspace-conventions.md` are retired — do not create them
