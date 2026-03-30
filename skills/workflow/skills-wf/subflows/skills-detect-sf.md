# Skills Detect Subflow

Monitors the active session for patterns that suggest a missing skill. When a gap
is identified, proposes the new skill and presents three options. Never re-proposes
a declined skill within the same session.

This is a background protocol — it runs continuously alongside all other work.

## Detection Triggers

Activate when any of the following conditions are met:

1. The same ad-hoc, multi-step process appears 2+ times in a session with no matching active skill
2. The user describes a workflow and asks the model to execute it repeatedly
3. The user references a tool or technology not covered by any active skill
4. The user explicitly asks: "is there a skill for X?" or "can you save this as a skill?"

Before proposing: run `skillforge ls` to confirm no skill with that name already exists.

## Proposal Format

```
Potential new skill: <name> (<type>) — <one-line description>.
Create now / Defer / Decline?
```

- `name`: lowercase, hyphen-separated, **with the required type suffix** — `-sme` for sme-persona, `-wf` for workflow (e.g. `postman-runner-wf`, `cloud-run-deployer-wf`, `api-standards-sme`)
- `type`: one of `sme-persona`, `workflow`
- Description: what the skill does, not how — max 15 words
- Propose a maximum of one skill per distinct pattern — do not flood the user

## Response Handling

### Create now
Load `skills-create-sf.md` and begin from Step 1 with the proposed name and type
pre-filled.

### Defer
Write a plan stub to:
`${SKILLFORGE_DIR}/memory/deferred-skills/<name>-YYYY-MM-DD.md`

```markdown
---
skill-name: <name>
skill-type: sme-persona | workflow
status: deferred
deferred-on: YYYY-MM-DD
---

# Deferred Skill: <name>

## Description
<one-line description>

## Trigger Context
<one paragraph: what the user was doing when this skill was identified>

## Implementation Steps
1. Invoke skills-wf → load skills-create-sf.md
2. Select template based on type
3. Define SKILL.md content based on trigger context above
4. Create memory stub via memory-wf
5. Run: skillforge activate <name>
```

Confirm to user: `Deferred plan saved: ${SKILLFORGE_DIR}/memory/deferred-skills/<name>-YYYY-MM-DD.md`

### Decline
Acknowledge: `Understood. Skill proposal suppressed for this session.`

Do not re-propose this skill for the remainder of the session.
Do not persist the decline — it is session-scoped only.
