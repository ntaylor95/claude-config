# Plan: GCP Seller Intelligence Terraform Infrastructure

## Context
This is a greenfield repository for provisioning Google Cloud resources that power a
ShipStation seller intelligence pipeline. The pipeline stores seller recommendation data
in BigQuery, exports it to a GCS bucket on a daily schedule, and exposes the bucket to
AWS via HMAC keys so AWS-side services can consume the data.

The GCP project (`shipstation-seller-intelligence`) already exists — Terraform only
provisions resources inside it.

---

## File Structure to Create

```
terraform/
├── main.tf               # Provider, required_providers, locals (project number, labels)
├── variables.tf          # All input variables
├── bigquery.tf           # BigQuery dataset + seller_recommendations table
├── storage.tf            # GCS export bucket + bucket IAM bindings
├── scheduled_query.tf    # BigQuery Data Transfer Service scheduled export
├── iam.tf                # Project-level IAM for DTS service agent + SA
├── hmac_keys.tf          # HMAC key for the existing service account
├── outputs.tf            # hmac_access_id, hmac_secret, bucket_name, dataset_id
└── sql/
    ├── export_query.sql          # EXPORT DATA OPTIONS ... AS SELECT placeholder
    └── create_seller_recommendations.sql  # CREATE TABLE DDL placeholder
```

---

## Resource Details

### 1. BigQuery Dataset (`bigquery.tf`)
- **Yes, a dataset is required** — BigQuery tables must live in a dataset
- Resource: `google_bigquery_dataset.seller_intelligence`
- Dataset ID: `seller_intelligence`
- Location: `US` (multi-region)
- Access blocks: OWNER for SA email, projectOwners/Readers/Writers

### 2. BigQuery Table (`bigquery.tf`)
- Resource: `google_bigquery_table.seller_recommendations`
- Table ID: `seller_recommendations`
- Schema: Terraform JSON schema block (placeholder columns) — user replaces after scaffolding, OR user can provide a `.sql` DDL later
- `deletion_protection = true`
- `lifecycle { ignore_changes = [schema] }` so manual BigQuery schema changes don't break Terraform

### 3. GCS Bucket (`storage.tf`)
- Resource: `google_storage_bucket.exports`
- Name: `shipstation-seller-intelligence-exports`
- Location: `US`
- `uniform_bucket_level_access = true`
- Lifecycle rules: STANDARD → NEARLINE at 90 days → COLDLINE at 365 days
- IAM bindings:
  - SA email → `roles/storage.objectAdmin` (HMAC owner reads/writes)
  - BigQuery DTS service agent → `roles/storage.objectAdmin` (to write export files)

### 4. BigQuery Scheduled Export (`scheduled_query.tf`)
- Resource: `google_bigquery_data_transfer_config.export_to_gcs`
- Mechanism: **`data_source_id = "scheduled_query"`** using the `EXPORT DATA OPTIONS` SQL statement — cleanest native approach, no extra compute needed, Parquet supported directly
- Schedule variable: `every day 02:00` (daily, set as default)
- SQL loaded via `templatefile()` from `sql/export_query.sql`, with template vars: project_id, dataset_id, table_id, bucket_name, path_prefix
- Enables `bigquerydatatransfer.googleapis.com` via `google_project_service`
- Depends on: DTS IAM grants and bucket IAM grants

### 5. IAM (`iam.tf`)
- **BigQuery DTS service agent** (`service-{number}@gcp-sa-bigquerydatatransfer.iam.gserviceaccount.com`):
  - `roles/bigquery.dataViewer` (project-level)
  - `roles/bigquery.jobUser` (project-level)
- **Existing SA**:
  - `roles/bigquery.dataViewer` (project-level, for verification)
- Bucket-level bindings live in `storage.tf` (not here)
- Project number fetched via `data "google_project" "current" {}`

### 6. HMAC Keys (`hmac_keys.tf`)
- Resource: `google_storage_hmac_key.service_account`
- References `var.service_account_email` (required variable, no default)
- `lifecycle { prevent_destroy = true }`
- **Note**: The HMAC `secret` is only available at creation time and lives in Terraform state — use an encrypted remote backend (GCS backend commented out in `main.tf` as a reminder)

### 7. Outputs (`outputs.tf`)
| Output | Sensitive | Purpose |
|--------|-----------|---------|
| `hmac_access_id` | false | S3-compatible access key ID |
| `hmac_secret` | **true** | S3-compatible secret — store in AWS Secrets Manager immediately |
| `gcs_bucket_name` | false | Bucket name for AWS consumer |
| `gcs_bucket_url` | false | Full GCS URL |
| `bigquery_dataset_id` | false | For reference |
| `bigquery_table_id` | false | For reference |
| `scheduled_query_name` | false | DTS config name |

---

## Variables Summary (`variables.tf`)

| Variable | Default | Required |
|----------|---------|----------|
| `gcp_project_id` | `shipstation-seller-intelligence` | — |
| `gcp_region` | `us-central1` | — |
| `environment` | `prod` | — |
| `bigquery_dataset_id` | `seller_intelligence` | — |
| `bigquery_dataset_location` | `US` | — |
| `gcs_bucket_name` | `shipstation-seller-intelligence-exports` | — |
| `gcs_bucket_location` | `US` | — |
| `service_account_email` | *(none)* | **Yes — user provides in tfvars** |
| `export_schedule` | `every day 02:00` | — |
| `export_path_prefix` | `seller-recommendations` | — |
| `table_partition_field` | `created_at` | — |

---

## SQL Placeholder Files

### `sql/export_query.sql`
Placeholder `EXPORT DATA OPTIONS` statement in Parquet format with SNAPPY compression.
Template vars: `${bucket_name}`, `${path_prefix}`, `${project_id}`, `${dataset_id}`, `${table_id}`.
User replaces `SELECT *` with their actual export SQL.

### `sql/create_seller_recommendations.sql`
Reference-only DDL file (not executed by Terraform, used for documentation/manual runs).
Placeholder columns with partitioning on `created_at`.

---

## Implementation Notes

- **Two-agent execution**: A GCP Infrastructure agent writes the Terraform HCL; a SQL Writer agent writes and validates the two SQL files
- **No project creation** — only resource provisioning
- **Terraform provider**: `hashicorp/google ~> 5.20.0`
- **State**: Remote GCS backend block included but commented out; user should uncomment and configure before running in CI
- **`terraform.tfvars`**: Not committed (in `.gitignore`); user creates locally with `service_account_email`

---

## Verification

1. `cd terraform && terraform init`
2. Create `terraform.tfvars` with `service_account_email = "your-sa@..."` and optionally override schedule
3. `terraform validate` — should pass with no errors
4. `terraform plan` — review all 10–12 resources before applying
5. After `terraform apply`:
   - Confirm dataset/table visible in BigQuery console
   - Confirm bucket created in GCS console
   - Confirm DTS scheduled query visible under BigQuery → Scheduled queries
   - Capture HMAC secret: `terraform output -raw hmac_secret`
6. Optionally trigger DTS job manually to verify Parquet files appear in bucket
