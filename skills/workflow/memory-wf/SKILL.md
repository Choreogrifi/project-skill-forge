---
name: memory-wf
description: Detects memory-change intent in user input and updates the correct workspace memory file with explicit user approval. Invoke directly or trigger automatically from natural language patterns.
metadata:
  skill-type: workflow
  version: "1.0"
  memory-file: workflow/memory.md
  disable-model-invocation: true
---

# Memory

Detects when the user wants to record, update, or remove a fact from workspace memory.
Identifies the correct target file, proposes the exact change as a before/after diff,
and writes only on explicit approval.

## Trigger Patterns

Activate when user input contains phrases such as:
- "remember that...", "note that...", "going forward..."
- "update my memory / profile / preferences..."
- "forget...", "remove from memory...", "I no longer..."
- "I prefer...", "from now on...", "always...", "never..."

## Workflow

### 1. Classify the Memory

Determine what type of fact the user wants to record:

| User Intent | Target File |
|---|---|
| SME domain knowledge or standard | `${SKILLFORGE_DIR}/memory/sme/<skill-name>.md` |
| Workflow-specific context or convention | `${SKILLFORGE_DIR}/memory/workflow/<skill-name>.md` |

If the target file is ambiguous, ask the user to confirm the category before proceeding.

### 2. Read the Target File

Read the full content of the target file before drafting any change.

### 3. Propose the Change

Present an explicit before/after diff:

```
Target: ${SKILLFORGE_DIR}/memory/workflow/<skill-name>.md

BEFORE (line N):
  - <existing content>

AFTER:
  - <existing content>
  - <new entry>

Approve? (yes / no)
```

For removals, archive the line rather than deleting it:
```
<!-- archived: YYYY-MM-DD: <reason> --> - <old content>
```

### 4. Write on Approval

- Write the change to the target file on explicit "yes"
- Update the `last-updated` field in the frontmatter to today's date
- Confirm the write: `"Memory updated: <target file>"`

## Constraints

- Never write without showing the diff and receiving explicit approval
- Never delete memory content — archive it with a comment instead
- If the target file does not exist, propose creating it with the correct frontmatter
  schema before writing any content
- Maximum body length per memory file: 40 lines — if exceeded, propose splitting into
  two scoped files

## Memory Scaffolding Operations

Invoke when the user asks to scaffold, list, view, or archive memory files (rather
than update their content).

### Create Memory Stub

Load subflow `subflows/memory-create-sf.md`.

### List Memory Files

Load subflow `subflows/memory-audit-sf.md`.

### Toggle System Skills

Load subflow `subflows/memory-system-toggle-sf.md`.

## Subflows

| File | Load when |
|---|---|
| `subflows/memory-create-sf.md` | User wants to scaffold a new memory file |
| `subflows/memory-audit-sf.md` | User wants to list, view, or archive memory files |
| `subflows/memory-system-toggle-sf.md` | User wants to enable or disable system skill activation |
