# Skills Audit Subflow

Audits skill content quality — frontmatter correctness, lean structure, and memory
file health. Symlink and state invariant issues are delegated to the `skillforge` CLI.

## Workflow

### 1. Discover Skills

Use Glob with pattern `${SKILLFORGE_DIR}/skills/*/SKILL.md` (across sme, workflow, system subdirs). For each result, extract the skill name and type from the directory path.

### 2. Run Checks

Read each `SKILL.md` and collect all failures before reporting. Run every check
regardless of earlier failures.

**Naming convention checks:**
- Directory base name does not end with `-sme` or `-wf` → `NAMING VIOLATION: expected <base>-sme or <base>-wf`

**Frontmatter checks:**
- `name` field value does not match directory name → `NAME MISMATCH`
- `description` field is absent or empty → `MISSING DESCRIPTION`
- `disable-model-invocation: true` absent → `MISSING DMI FLAG`
- `metadata.skill-type` field is absent → `MISSING SKILL-TYPE`
- `metadata.memory-file` field is absent → `MISSING MEMORY-FILE` (warn — optional but expected)

**Lean checks:**
- Any contiguous block of non-heading, non-code prose exceeds 10 lines → `BLOATED SECTION` (flag the heading)
- Long lists, lookup tables, or embedded specs present with no `## References` section → `MISSING REFERENCES SECTION`

**Memory checks:**
- For each skill with a `memory-file` field: check `${SKILLFORGE_DIR}/memory/<memory-file>` exists → if not → `MISSING MEMORY FILE`
- For each `.md` file under `${SKILLFORGE_DIR}/memory/` (exclude `deferred-skills/` and `*.archived.md`): verify at least one active skill's `memory-file` references it → if not → `ORPHANED MEMORY FILE`
- For each memory file: check `last-updated` frontmatter field — if older than 90 days → `STALE MEMORY FILE`

**Deferred skill checks:**
- Use Glob `${SKILLFORGE_DIR}/memory/deferred-skills/*.md`
- For each, read the `deferred-on` frontmatter field
- If older than 30 days → `DEFERRED SKILL OVERDUE: <skill-name>`

### 3. Delegate State Issues

Do not attempt to fix symlinks or rename directories. After presenting the report:

```
For state and validation violations, run: skillforge audit
```

### 4. Report

Present a full summary table:

```
Skill           | Check                        | Status
--------------- | ---------------------------- | ------
```

Flag content and memory issues for manual review. Do not auto-edit skill bodies
or memory files.

## Guidelines

- Read-only — this subflow never writes files.
- Invariant enforcement belongs exclusively to `skillforge audit`.
- Surface deferred skill reminders as informational — do not block the report on them.
