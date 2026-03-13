# GCP Terraform / OpenTofu with Spacelift CI/CD

## Purpose
Best practices for writing GCP Terraform/OpenTofu with Spacelift CI/CD. Covers IAM scoping,
lifecycle guards, file structure conventions, Spacelift patterns, and a pre-PR checklist.
Use this skill when writing new GCP infrastructure, reviewing existing Terraform code, or
preparing a PR for a GCP stack managed by Spacelift.

## When to Use This Skill
- Writing new GCP Terraform or OpenTofu modules
- Reviewing a Terraform PR for correctness and security
- Checking existing infrastructure code against best practices before opening a PR
- Debugging IAM permission errors in GCP stacks
- Structuring a new Terraform stack for Spacelift deployment

## Inputs
The user will typically provide one or more of:
- A `.tf` file or directory to review
- A description of the infrastructure they are building
- A PR diff or summary of recent changes

If no input is provided, prompt the user to share the relevant `.tf` files or describe
what they are building.

## Steps

### 1. Check Lifecycle Guards on Credential Resources
Scan all resources that represent one-time or hard-to-rotate credentials:
- `google_storage_hmac_key`
- `google_service_account_key`
- Any resource whose destruction would break a downstream system

Each must have:
```hcl
lifecycle {
  prevent_destroy = true
}
```

`prevent_destroy` defaults to `false`. The "not dynamic in OpenTofu/Terraform" justification
is incorrect — it is a valid static boolean in both tools. Flag any credential resource
missing this block as a blocker.

### 2. Audit IAM Binding Scope
For every `google_project_iam_member` or `google_project_iam_binding`, ask: can this be
scoped to a narrower resource?

**Wrong — project-wide when dataset scope is sufficient:**
```hcl
resource "google_project_iam_member" "scheduler_bq_data_viewer" {
  project = var.gcp_project_id
  role    = "roles/bigquery.dataViewer"
  member  = "serviceAccount:${google_service_account.scheduler_sa.email}"
}
```

**Right — dataset-scoped:**
```hcl
resource "google_bigquery_dataset_iam_member" "scheduler_bq_data_viewer" {
  dataset_id = google_bigquery_dataset.seller_intelligence.dataset_id
  role       = "roles/bigquery.dataViewer"
  member     = "serviceAccount:${google_service_account.scheduler_sa.email}"
}
```

Exception: `roles/bigquery.jobUser` legitimately requires project scope.

For GCS access, use the minimum viable role:
- `roles/storage.objectCreator` — write new files only
- `roles/storage.objectAdmin` — overwrite or delete existing paths
- `roles/storage.objectViewer` — read-only consumers (e.g. HMAC SA for cross-cloud access)

### 3. Check IAM `depends_on` for Conditionally-Enabled APIs
When a resource depends on a `google_project_service` being fully propagated, the IAM
binding also needs `depends_on` — not just the resource that consumes it:

```hcl
resource "google_project_iam_member" "scheduler_bq_data_viewer" {
  depends_on = [google_project_service.bigquery_data_transfer]
  project    = var.gcp_project_id
  role       = "roles/bigquery.dataViewer"
  member     = "serviceAccount:${google_service_account.scheduler_sa.email}"
}
```

Flag any IAM binding for a service-dependent resource that is missing `depends_on`.

### 4. Prefer Dedicated Service Accounts over Managed Service Agents
Flag any use of GCP-managed service agents
(e.g. `service-{number}@gcp-sa-bigquerydatatransfer.iam.gserviceaccount.com`).
These are opaque, harder to audit, and grant broader implicit permissions.
Recommend a dedicated `google_service_account` with explicit IAM bindings instead.

### 5. Verify File Structure
A well-structured GCP Terraform stack uses these filenames:

```
terraform.tf         # required_version + required_providers
provider.tf          # provider "google" block
locals.tf            # locals {} blocks only — NOT named data.tf
variables.tf         # variable declarations
outputs.tf           # output declarations
data.tf              # data "google_..." sources only (if the stack uses them)
<resource>.tf        # one file per logical resource group (bigquery.tf, storage.tf, etc.)
```

Specifically flag:
- `locals {}` blocks placed inside a file named `data.tf` — this creates confusion because
  engineers will look in `data.tf` for data sources and either add to it incorrectly or
  create a duplicate file
- Multiple logical resource groups merged into a single large file

### 6. Check SQL Files Used as templatefile() Templates
For any `.sql` file referenced by a `templatefile()` call:
- Never write `${...}` in SQL comments — Terraform parses the entire file including comments
- Every `${var}` in the SQL body must have a corresponding key in the `templatefile()` vars map
- Escape any literal `${...}` string as `$${...}`

**Wrong:**
```sql
-- Terraform substitutes ${variable} placeholders at plan time
SELECT ${column_name} FROM ...
```

**Right:**
```sql
-- Terraform substitutes template placeholders at plan time
SELECT ${column_name} FROM ...
-- and column_name is declared in the templatefile() vars map
```

### 7. Audit Spacelift Patterns

**State management:** Spacelift manages state natively (`manage_state = true`). Remove any
commented-out `backend "gcs"` blocks — they are obsolete and misleading.

**tfvars in git:** Committing `*.tfvars` is acceptable for Spacelift when:
- The file contains only non-sensitive values (environment names, dataset IDs, resource names)
- The file is referenced by the Spacelift stack's `--var-file` argument
- A comment in the file explicitly states no secrets should be added

Do not remove `*.tfvars` from `.gitignore` without documenting the boundary. Sensitive values
(API keys, passwords, project credentials) must always go into Spacelift stack variables
set in the UI, not in committed var files.

**Deletion protection:** Use conditional deletion protection so dev stacks can be destroyed freely:
```hcl
deletion_protection = var.environment == "prod"
```

**Labels:** Include a `managed_by` label that reflects the actual deployment system:
```hcl
locals {
  common_labels = {
    environment = var.environment
    project     = "your-project-name"
    managed_by  = "spacelift"
  }
}
```

### 8. Verify Provider Version Pinning
Production stacks must pin to patch level:

```hcl
# Correct for production — allows 5.20.x only
version = "~> 5.20.0"

# Acceptable only if intentional and documented — allows 5.21, 5.22, etc.
version = "~> 5.20"
```

The Google provider has historically introduced breaking changes in minor versions.
Flag any production stack using a minor-level pin (`~> x.y`) without a documented
reason, and any stack using an unpinned or overly broad version constraint.

### 9. Run the Pre-PR Checklist
Before marking work complete, confirm every item below:

- [ ] Credential resources (HMAC keys, SA keys) have `lifecycle { prevent_destroy = true }`
- [ ] IAM bindings use the narrowest resource scope available (dataset/bucket, not project)
- [ ] IAM bindings that depend on an API being enabled have `depends_on = [google_project_service.xxx]`
- [ ] Dedicated SAs used instead of GCP managed service agents where possible
- [ ] `locals {}` blocks live in `locals.tf`, not `data.tf`
- [ ] No `${...}` in SQL comments inside templatefile() templates
- [ ] Sensitive values are in Spacelift stack variables, not committed tfvars
- [ ] `deletion_protection` is conditional on environment for dev/prod stacks
- [ ] Provider version is pinned to patch level (`~> x.y.z`) for production stacks
- [ ] All files end with a newline character

Report each failing item with the file name, line reference if available, and the
corrected code block.

## Output
Produce one of the following depending on the task:

- **Review mode:** A structured report grouped by severity (Blocker / Warning / Convention).
  Each finding includes the file, the problematic code, and the corrected version.
- **Write mode:** Terraform code that satisfies all rules in this skill, with inline comments
  explaining non-obvious choices (e.g. why a particular IAM scope was chosen).
- **Checklist mode:** The pre-PR checklist with pass/fail status for each item and
  actionable fixes for any failures.

## Standards and References
This skill is derived from a Google Cloud Sr. Engineer PR review of a real GCP Terraform
stack. It covers Terraform and OpenTofu equally — both tools are subject to the same rules.

Relevant project-level rules (when working in the gcs-seller-intelligence repo):
- `.claude/rules/terraform-templatefile-sql.md` — full detail on templatefile() SQL rules
- `CLAUDE.md` — project constraints including schema change policy and HMAC SA permissions
