---
skill: gitlab-wf
skill-type: workflow
description: GitLab operations context — CLI tools, approval gate, MR conventions
last-updated: 2026-03-30
---

## Workflow Context

- Uses `git` for local operations, `glab` for GitLab platform operations
- Every proposed action is presented as a dry-run summary before execution
- Nothing executes without explicit user approval
- References: `${SKILLFORGE_DIR}/skills/workflow/gitlab-wf/references/gitlab-operations.md`

## MR Standards

- MR title: conventional commit style (`feat:`, `fix:`, `chore:`)
- MR body: summary, test plan, checklist
- Squash merge preferred for feature branches
- Pipeline must pass before merge — no pipeline bypass
