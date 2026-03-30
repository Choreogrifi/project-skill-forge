# SkillForge Migration Review
**Date:** 2026-03-30
**Branch stabilised:** main @ 4dedfc2 (PR #8)
**Scope:** migrations v1–v4 + incident recovery (2026-03-27 → 2026-03-30)
**Diagram:** [skillforge-architecture.mmd](skillforge-architecture.mmd)

---

## 1. Security

### Findings

| # | Area | Finding | Severity |
|---|---|---|---|
| S1 | `scripts/install.sh` | Symlink creation with `ln -s` does not verify the target is within an expected base path. A user-supplied `LLM_DIR` value could redirect the symlink to an attacker-controlled location. | Medium |
| S2 | `scripts/skillforge.sh` | Shell script processes user-supplied arguments. If any argument is passed to `eval` or used in an unquoted shell expansion, command injection is possible. Needs `shellcheck` run. | Medium |
| S3 | `scripts/hooks/pre-commit` | Hook runs `check-skill-names.sh` on every commit. If the script reads file paths without quoting, file names with spaces or special characters will cause injection. | Low |
| S4 | `scripts/install.sh` | `curl`-based install pattern (common for this type of project) should be documented with checksum verification. No integrity check is currently surfaced in README. | Low |
| S5 | `templates/persona/model.md` | Persona template is user-editable and loaded into Claude context. No sanitisation boundary exists. Users should be warned not to store secrets or PII in persona files. | Informational |
| S6 | `.github/workflows/validate.yml` | CI runs `bats` and eval scripts. If test fixtures reference external resources, a compromised fixture could exfiltrate CI secrets. Review fixture isolation. | Low |

### Recommendations
- Run `shellcheck` across all `.sh` files and enforce it in CI.
- Add path canonicalisation and base-path assertion in `install.sh` before creating symlinks.
- Document curl install checksum verification in `README.md` and `docs/getting-started.md`.
- Add a note to `templates/persona/model.md` warning against storing credentials.

---

## 2. Efficiency

### What Works Well
- **Scoped memory loading** — memory is loaded per skill-type (`sme/`, `workflow/`, `shared/`) not globally. Token overhead per activation is minimal.
- **Once-per-session `identity.md`** — the shared identity file is loaded once, not on every sub-skill activation. Correct.
- **Subflow pattern** — parent skills stay small; detail is deferred to subflows loaded on demand. This is the right architecture for token budget management.
- **Skill lifecycle suffixes** (`.active`, `.review`, `.deactivated`, `.decommissioned`) — prevents dead skill files from polluting context without deleting history.

### Gaps
| # | Area | Finding |
|---|---|---|
| E1 | `memory/shared/` | `symlink-registry.md` is the only shared file after `identity.md` and `workspace-conventions.md` were removed. Consider whether shared context is now too thin for cold-start sessions. |
| E2 | Skill activation routing | No automated routing guard exists in `skillforge.sh` to prevent invoking a non-`.active` skill. Governance relies entirely on Claude following CLAUDE.md rules — not enforced at the CLI layer. |
| E3 | `check-skill-names.sh` | Validates naming conventions but does not validate frontmatter completeness (`skill-type` field, `description` field). A skill missing `skill-type` silently becomes `.review` per governance rules — this should be a CI failure, not a silent fallback. |
| E4 | Memory files for `system/` type | `memory/system/` was deleted entirely (manage-skills, memory-manager, skill-detector memory files removed). System-type skills now have no memory backing. If system skills are re-introduced they will have no context to load. |

---

## 3. Productivity Magnification

### What Works Well
- **sme/ + workflow/ hierarchy** replaces the flat `*.active` pattern. Discovery is now structural — navigating the repo reveals capability without reading every SKILL.md.
- **Subflow decomposition** enables Claude to execute complex multi-step processes (e.g. `git-wf` covering branch, clone, commit, conflict, tag, repo-create, repo-rename) without loading the entire context upfront.
- **Templates** (`templates/memory/`, `templates/persona/`, `templates/workflow/`) reduce onboarding time for new skills from scratch to a fill-in-the-blanks exercise.
- **Tests + evals** (`tests/evals/`, `tests/run_evals.sh`, `tests/test_skillforge.bats`) provide confidence that skill renames and consolidations have not broken behaviour.
- **`skillforge.sh` CLI** enables skill management without entering an LLM session — a significant productivity unlock.
- **Cross-referencing** (e.g. `terraform-wf` referencing `gcp-wf` for project discovery) avoids duplication and keeps each skill focused.

### Gaps
| # | Area | Finding |
|---|---|---|
| P1 | CLI skill invocation | `migration-v1.md` item 10: `skillforge claude add-new-skill` style direct invocation is not yet implemented. Current CLI manages the workspace but does not delegate to LLM skills directly from the shell. |
| P2 | Python wrapper | `migration-v1.md` item 10: Python wrapper for skills callable from the command line is not yet started. No `pyproject.toml`, `setup.py`, or `src/` tree exists. |
| P3 | Add-LLM workflow | `migration-v1.md` item 10: "Allow a user to add a LLM — this process must create all resources required" is not yet implemented as a guided subflow. `install.sh` handles installation but not multi-LLM registration. |
| P4 | Eval coverage | Current evals cover `architect-sme`, `git-sme`, `security-sme`. No evals exist for `gcp-sme`, `terraform-sme`, `skills-wf`, or `document-wf` — the highest-traffic skills in practice. |
| P5 | Markdown linter in CI | `migration-v2.md` item: markdown linter was listed as a requirement. `validate.yml` runs `check-skill-names.sh` and bats but no markdown linter (e.g. `markdownlint`). |
| P6 | `git-commit-wf` isolation | `git-commit-wf` is a standalone skill not nested under `git-wf/subflows/`. This breaks the expected discovery path and may cause invocation inconsistency. Revisit whether it should become `git-wf/subflows/git-commit-sf.md`. |

---

## 4. General Gaps & Proposed Solutions

| # | Gap | Proposed Solution | Priority |
|---|---|---|---|
| G1 | `docs/` not fully regenerated post-migration | `docs/skill-catalog.md` exists but may not reflect the new `sme/` + `workflow/` folder structure. All `docs/` pages should be reviewed and regenerated before GitHub Pages publish. | High |
| G2 | No threat model or security policy | Add `SECURITY.md` at repo root documenting responsible disclosure, install integrity verification, and scope of shell execution. | Medium |
| G3 | `memory/system/` deleted with no replacement | System-type skills (`manage-skills`, `skill-detector`) were folded into `skills-wf` and `memory-wf` but their memory context was deleted. The `skills-wf` memory file (`memory/workflow/skills.md`) should absorb the critical parts. | Medium |
| G4 | No `.memory/` gitignore rule | The `.memory/` folder in the project root is a session/project-scoped memory store. It should be added to `.gitignore` to prevent accidental commit of ephemeral notes, or explicitly tracked with a documented intent. | Low |
| G5 | `RECOVERY-PLAN.md` + `migration-v*.md` loose files | Contextualised here and in `skillforge-architecture.mmd`. Safe to delete — all requirements are implemented or tracked as gaps above. | Done |
| G6 | `skillforge-chat-history.md` | 92 user messages from 2026-03-26 → 2026-03-29 captured for audit. No persistent value after this review. Safe to delete. | Done |
| G7 | Self-discovery loop | `migration-v4.md` item 4: skills should be able to self-discover improvements and raise them as PRs. The `skills-detect-sf.md` and `skills-propose-sf.md` subflows scaffold this, but the end-to-end loop (detect → plan → test → PR) is not validated end-to-end. | Medium |
| G8 | Persona template genericness | `migration-v2.md`: persona template must be generic and user-driven. `templates/persona/model.md` exists but `templates/persona/system-skills-always-on.md` and `system-skills-manual.md` are specific to Claude Code. Gemini or other LLM support requires additional persona variants. | Low |

---

## Summary

The migration-v1 through v4 requirements are **substantially implemented**. The recovery from the delete incident was clean, with `ccc6d4b` restoring all files and `8ad1ae9` completing the renaming and consolidation. The architecture is sound: scoped memory, subflow decomposition, lifecycle governance, and a CLI layer are all in place.

The highest-priority next actions before publishing are:

1. Run `shellcheck` on all scripts and fix any injection paths (S2).
2. Regenerate `docs/` to reflect the new skill hierarchy (G1).
3. Implement CLI-to-LLM skill delegation (`skillforge claude <skill>`) (P1).
4. Add `markdownlint` to CI (P5).
5. Validate `skills-detect → skills-propose → PR` end-to-end (G7).
