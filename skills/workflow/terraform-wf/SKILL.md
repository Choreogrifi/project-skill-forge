---
name: terraform-wf
description: Guided workflow to scaffold and discover Terraform HCL in the current project. Analyses codebase infrastructure signals and optionally cross-references live GCP infrastructure. Invoke when adding a new module, resource, or workload; or when you need an infrastructure proposal derived from application code.
metadata:
  skill-type: workflow
  version: "1.0"
  memory-file: workflow/terraform.md
  disable-model-invocation: true
---

# Terraform

Covers two integrated workflows: **discovery** (analysing a codebase to infer required infrastructure) and **scaffolding** (generating HCL that conforms to project conventions). Both workflows may cross-reference live GCP state via `gcp-wf`.

---

## Workflow A — Scaffold New Terraform Resources

Scaffolds new Terraform HCL that conforms to the project's established conventions. Reads existing state to understand what is already deployed before generating anything new. If no Terraform folder exists, bootstraps one using Workflow B to analyse the codebase and optionally `gcp-wf` to cross-reference live GCP infrastructure.

### A.1. Detect Terraform Folder

Check if a `./terraform` directory exists in the current working directory.

**If `./terraform` exists:**
- Run `terraform workspace show` from within `./terraform` to identify the active environment.
- Look for a state file at `./terraform/terraform.tfstate.d/<env>/terraform.tfstate` where `<env>` is the value returned by the command.
- If found, read it and extract the list of managed resource types and names.
- Present a brief as-is summary:
  ```
  As-is (env: <env>)
  ──────────────────
  <resource_type>: <resource_name>
  ...
  ```
- Read `references/pwd-terraform-standards.md` to load the project conventions.
- Proceed to Step A.2.

**If `./terraform` does not exist:**
- Inform the user: "No terraform folder found. I'll bootstrap one."
- Create the `./terraform` directory.
- Run Workflow B to analyse the codebase and produce a proposed resource plan. Hold this output as the **discovery baseline**.
- Ask the user: "Would you like me to cross-reference a live GCP project to refine the proposal? (yes / no)"
  - If yes: invoke `gcp-wf`, which will prompt for the GCP project ID. Merge its inventory with the discovery baseline — resources already present in GCP are marked `[existing]`; new resources are marked `[new]`.
  - If no: continue with the discovery baseline alone; all proposed resources are marked `[new]`.
- Read `references/pwd-terraform-standards.md` to load the project conventions.
- Proceed to Step A.2 with the discovery baseline pre-populating resource selections and variable suggestions.

### A.2. Gather Requirements

Ask the user what they want to create. Collect all fields before proceeding:

```
What do you want to add?

  a) New workload (Cloud Run Job + Scheduler + IAM + Secrets)
  b) New foundation resource only (e.g. extra SA, subnet, secret group)
  c) New top-level module pair (foundation + workload)
  d) Something else — describe it

Enter a, b, c, or d:
```

For options a–c, ask follow-up questions to gather:
- `product_name` and `module_name` (if not already set in common.tfvars)
- Any domain-specific variables (e.g. BigQuery dataset/table, cron schedule, Redis size)
- Which environments this applies to (`dev` only, `dev + prd`, all)

For option d, ask the user to describe the resource(s) and intended responsibility, then map it to the closest standard pattern.

### A.3. Draft HCL

Generate the full HCL for all required files, strictly following `references/pwd-terraform-standards.md`:

- Apply the naming convention: `${product_name}-${module_name}-${resource_suffix}`
- Place infrastructure resources (APIs, IAM, VPC, storage, secrets) in `modules/foundation/`
- Place compute resources (Cloud Run, Scheduler) in `modules/workload/`
- Wire modules via root `main.tf` — workload receives foundation outputs as explicit vars
- Add `depends_on = [module.foundation]` in the workload module call
- Use `terraform.workspace` for environment-differentiated values (tier, CIDR, log level)
- Add or update `common.tfvars` with shared values; add per-env tfvars for `project_id` and schedules
- Ensure `variables.tf` and `outputs.tf` exist in every module and the root
- Use `for_each` + `toset()` for API enablement and secret groups
- No hardcoded credentials — all sensitive data via Secret Manager
- Add `lifecycle { ignore_changes = [...] }` for CI/CD-managed attributes

Present the full draft as a file tree with the content of each file:

```
Files to create / modify:
─────────────────────────
terraform/
  main.tf               [modified]
  variables.tf          [modified]
  common.tfvars         [modified]
  dev.tfvars            [new]
  modules/
    foundation/
      iam.tf            [new]
      ...
```

Show the complete HCL for each file.

### A.4. Confirm with User

```
Does this look correct? (yes / edit / cancel)
```

- `yes` → proceed to Step A.5
- `edit` → ask what to change, re-draft, and return to this step
- `cancel` → stop; no files are written

### A.5. Execute

On approval:

1. Write all files. For each file:
   - If it is a **new** file, write it in full.
   - If it is a **modified** file, apply only the additions/changes needed — do not overwrite unrelated content.

2. Run `terraform init` from within `./terraform` to download providers and modules.

3. Run `terraform workspace select <env>` (where `<env>` is the workspace identified in Step 1, or `dev` for a new setup) to ensure the correct workspace is active.

4. Confirm completion:

```
Done.

Files written:
  <list of paths>

Workspace ready: <env>
terraform init and workspace selection complete — ready for plan and apply.
```

---

## Workflow B — Discover Infrastructure from Code

Reads the application codebase in the current working directory and infers what GCP infrastructure the application requires. Produces a structured resource proposal. Does not write any files — output only.

### B.1. Scan

Scan broadly before concluding — check all of: `Dockerfile`, `package.json` / `requirements.txt` / `go.mod`, `.env.example`, `docker-compose.yml`, CI pipeline files, source imports, and any existing config files.

Load subflow `subflows/terraform-scan-sf.md` to perform the scan.

### B.2. Map

Map each signal to a GCP resource type and module layer (foundation or workload) using `references/signal-to-resource-map.md`.

### B.3. Propose

Return a structured proposal:

```
Terraform Discovery Report
──────────────────────────
Runtime:       <detected runtime, e.g. Node.js 20, Python 3.11>
Trigger:       <how the workload is invoked, e.g. HTTP, Pub/Sub, Cloud Scheduler>

Foundation resources:
  [high]   google_project_service         — APIs: run, secretmanager, bigquery, ...
  [high]   google_service_account         — worker SA (least-privilege)
  [medium] google_vpc_network             — custom VPC (Redis/private service detected)
  [medium] google_vpc_access_connector    — serverless VPC access
  [medium] google_redis_instance          — cache client detected
  [high]   google_secret_manager_secret   — env vars: <list of detected secret names>
  [low]    google_artifact_registry_repository — Dockerfile present

Workload resources:
  [high]   google_cloud_run_v2_job        — batch/job pattern detected
  [medium] google_cloud_scheduler_job     — cron config detected

Suggested variables:
  product_name  = "<inferred from repo/package name>"
  module_name   = "<inferred from entry point or service name>"

Ambiguous signals:
  <signal> → could be <resource A> or <resource B> — needs clarification
```

---

## Guidelines

- **Never act silently:** Always present a summary and wait for approval before writing files.
- **Respect scope:** Only generate what the user explicitly requested — do not add unrequested resources.
- **Standards are non-negotiable:** All output must conform to `references/pwd-terraform-standards.md`. Never deviate from the naming convention, module separation, or IAM least-privilege pattern.
- **No hardcoded credentials:** Reject any request to inline secrets; redirect to Secret Manager.
- **State is read-only:** Read state files to understand as-is; never modify or delete them.
- **No invention:** Every proposed resource must trace back to at least one code signal.
- **No credentials in proposals:** If secret values are visible in code, flag them as a security-sme issue; do not include them in the proposal.

## Subflows

| File | Load when |
|---|---|
| `subflows/terraform-scan-sf.md` | Codebase scan for infrastructure signals (Workflow B) |

## References

- `references/pwd-terraform-standards.md` — project conventions (naming, module structure, IAM, secrets, workspace config); read during Step A.1
- `references/signal-to-resource-map.md` — mapping of code signals to GCP resource types; read during Workflow B
- `references/terraform-standards.md` — general Terraform standards reference

## Related Skills

- `terraform-sme` — SME for Terraform architecture decisions and module design
- `gcp-sme` — SME for GCP service selection and architecture
- `gcp-wf` — GCP project inventory; invoked during bootstrap to cross-reference live infrastructure
