---
skill: terraform-wf
skill-type: workflow
description: Context for Terraform scaffolding and codebase-based infrastructure discovery — conventions, state, module structure, signal mapping
last-updated: 2026-03-30
---

## Workflow Context

- Reads existing `./terraform` folder before generating anything — never create HCL blind
- Invokes codebase scan (Workflow B) if no `./terraform` folder exists
- Optionally invokes `gcp-wf` to cross-reference live GCP state
- All proposed resources must be shown to the user before writing
- Codebase scan is read-only — never writes files during discovery

## Module Structure

- Separate modules: `foundation` (IAM, APIs, VPC, secrets, storage) and `workload` (compute, scheduling)
- Never mix foundation and workload resources in the same module
- Remote GCS state required — no local state files in any environment

## Key Conventions

- Provider block: `google` and `google-beta`, pinned version
- Backend block: GCS with prefix per environment
- Variable files: `variables.tf`, `outputs.tf`, `main.tf`, `versions.tf` — no monolithic files
- Naming: all resources follow `<team>-<env>-<resource-type>` pattern

## Signal Sources (codebase scan)

- Environment variable names → Secret Manager secrets
- Docker base images → Cloud Run or GKE workload type
- Database clients → Cloud SQL or Firestore
- Pub/Sub imports → topic and subscription resources
- GCS client imports → storage bucket resources

## Design Decision Trigger

If a resource requires an architectural choice (e.g., Cloud Run vs GKE, Pub/Sub vs Cloud Tasks),
pause and invoke the `architect-sme` before proceeding.
