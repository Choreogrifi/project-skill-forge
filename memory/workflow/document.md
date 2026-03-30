---
skill: document-wf
skill-type: workflow
description: Router context — operation routing and subflow delegation for all document operations
last-updated: 2026-03-30
---

## Workflow Context

- Collects document subject from user before selecting operation and template
- Routes to subflows: `document-adr-sf`, `document-readme-sf`, `document-audit-sf`, `document-update-sf`, `document-extend-sf`
- Confirms file location with user before any write
- All file writes require explicit user approval of the target path
- Diagram requests are delegated to `mermaid-wf`

## Operation Routing

- New README → `document-readme-sf`
- New diagram → `mermaid-wf`
- New ADR → `document-adr-sf`
- Review quality → `document-audit-sf`
- Update existing section → `document-update-sf`
- Add new section → `document-extend-sf`

## Review Dimensions

- Structure and completeness
- Clarity and readability
- Technical accuracy
- Discoverability (for public-facing docs)
- Consistency with workspace documentation standards

## README Layout

- Follows 2026 README layout standard (see sme/document.md for section order)
- Confirms target path before writing
