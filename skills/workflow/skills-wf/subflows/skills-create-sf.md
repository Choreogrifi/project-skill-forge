# Skills Create Subflow

Guides the user through authoring a new skill — collecting name, type, structure, and
content — then writes all files. Activation is always delegated to the `skillforge` CLI.

## Naming Standard

All skill names **must** end with a type suffix:

| Suffix | Applies to | Example |
|--------|------------|---------|
| `-sme` | `sme-persona` type skills | `security-sme`, `architect-sme` |
| `-wf`  | `workflow` type skills | `git-commit-wf`, `terraform-wf` |

The suffix makes the skill's role immediately visible in directory listings and git diffs.
Any skill directory that does not follow this pattern fails the pre-commit naming check.

## Workflow

### 1. Collect Name and Type

```
What type of skill?

  1. sme-persona  — domain expertise and standards (e.g. architect-sme, security-sme)
  2. workflow     — step-by-step automation (e.g. git-commit-wf, terraform-wf)

Enter 1 or 2:
```

Then:

```
Skill base name (lowercase, hyphen-separated, without the type suffix):
```

Append the required suffix automatically:
- Type 1 selected → final name = `<base>-sme`
- Type 2 selected → final name = `<base>-wf`

Show the resolved name and confirm before proceeding:

```
Skill will be created as: <final-name>
Continue? (yes / cancel)
```

Validate: final name matches `^[a-z][a-z0-9_-]+-(sme|wf)$`. Run `skillforge ls` to confirm no skill with that name already exists.

### 2. Determine Subflow File Structure

For `sme-persona`: skip — these skills are single-file by nature.

For `workflow`: ask:

```
Will this skill have multiple distinct operations?
If yes, list each operation name (one per line). These become subflow files under subflows/.
If no, all workflow steps stay in SKILL.md.

Examples: "create", "detect", "audit" → skills-create-sf.md, skills-detect-sf.md, skills-audit-sf.md
```

Collect the list of subflow names (or none).

### 3. Collect Description

```
One-line description (what it does and when to invoke it):
```

### 4. Collect Body Content

**sme-persona**: Collect in order — Focus area, Standards (bullet list),
Mandatory tasks (numbered list), Constraints (bullet list).

**workflow SKILL.md** (with subflows): Collect only the routing logic —
what each subflow does and when to load it.

**workflow SKILL.md** (single file): Collect full workflow steps — problem
statement, step names, guidelines.

**For each subflow file**: Collect separately — step names and descriptions, guidelines.

### 5. Collect Reference Files

```
Does this skill need reference files (lookup tables, specs, lists >10 lines)?
If yes, describe each one. (yes / no)
```

If yes: collect name and content for each. Write to `references/<name>.md` inside
the skill directory. Add a `## References` section listing each file and the
step/condition that triggers loading it.

### 6. Preview and Confirm

Display the generated `SKILL.md` and any subflow files. Ask:

```
Does this look correct? (yes / edit / cancel)
```

On `edit`: ask which section to change, collect the replacement, re-display.
On `cancel`: stop — nothing is written.

### 7. Write Files

All files are written to `${SKILLFORGE_DIR}/skills/` — never to the working directory.

Determine the target type directory from the skill type:
- `sme-persona` → `${SKILLFORGE_DIR}/skills/sme/<name>/`
- `workflow` → `${SKILLFORGE_DIR}/skills/workflow/<name>/`

1. Create the skill directory
2. Write `SKILL.md` using the appropriate template:
   - `sme-persona` → `templates/expertise-skill-template.md`
   - `workflow` → `templates/workflow-skill-template.md`
3. Write each subflow file under `subflows/` using `templates/workflow-file-template.md`
4. Write each `references/<file>.md` if collected in Step 5
5. Check `references/marketplace-overlap.md` — warn on any conflict
6. Confirm:
   ```
   Created: ${SKILLFORGE_DIR}/skills/<type>/<name>/
   Files:   SKILL.md<, subflow files, reference files>
   ```

### 8. Delegate Activation

```
Run: skillforge activate <final-name>
```

Do not create symlinks inline — the CLI owns that operation.
Offer to create a memory stub: ask the user to invoke `memory-wf` for that.

## Guidelines

- Never write skill files outside `${SKILLFORGE_DIR}/skills/`.
- A skill with a single operation keeps everything in `SKILL.md` — do not split prematurely.
- Reference rule: any content >10 lines needed for only one step belongs in `references/`, not inline.
- Subflow files have no frontmatter — they are markdown documents loaded lazily by the parent SKILL.md.
