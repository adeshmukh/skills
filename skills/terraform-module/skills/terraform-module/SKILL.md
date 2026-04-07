---
name: terraform-module
description: Generate a GCP Terraform module or infra directory with provider config, variables, outputs, tfvars examples, GCS backend, and .gitignore. Use when setting up Terraform infrastructure for a GCP project.
argument-hint: "<resource-type> [--name module-name] [--env dev|staging|prod]"
allowed-tools: Write, Read, Glob, Bash
---

# Terraform Module Generator (GCP)

Generate Terraform infrastructure files following established GCP conventions.

## Step 1 — Parse Arguments

| Parameter | Default | Description |
|-----------|---------|-------------|
| `resource-type` | (required) | GCP resource to scaffold. One of: `gcs`, `cloud-run`, `cloud-sql`, `vpc`, `iam`, `full-stack` |
| `--name` | derived from resource-type | Module directory name (kebab-case) |
| `--env` | `dev` | Environment label for tfvars and tags |

`full-stack` generates modules for Cloud Run + Cloud SQL + GCS together — a common deployment pattern.

## Step 2 — Determine Output Location

Check if an `infra/` directory exists in the current project:
- If yes: generate inside `infra/modules/<name>/`
- If no: create `infra/` at the project root with the full scaffolding

## Step 3 — Generate Root Infrastructure

If `infra/` does not already exist, create the root scaffolding:

```
infra/
├── versions.tf
├── variables.tf
├── main.tf
├── outputs.tf
├── backend.tf.example
├── terraform.tfvars.example
├── .gitignore
└── modules/
    └── <module>/
```

### versions.tf

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.35"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

### variables.tf (root)

```hcl
variable "project_id" {
  description = "The GCP project ID"
  type        = string
}

variable "region" {
  description = "GCP region"
  type        = string
  default     = "us-central1"
}
```

Add resource-specific variables that are passed through to modules.

### main.tf (root)

Instantiate the module(s):

```hcl
module "<module_name>" {
  source = "./modules/<module_name>"

  project_id = var.project_id
  # ... pass through variables
}
```

### outputs.tf (root)

Re-export key module outputs:

```hcl
output "<resource>_name" {
  description = "Name of the created resource"
  value       = module.<module_name>.<resource>_name
}
```

### backend.tf.example

```hcl
terraform {
  backend "gcs" {
    bucket  = "REPLACE_WITH_TF_STATE_BUCKET"
    prefix  = "terraform/state/<project-name>"
    project = "REPLACE_WITH_PROJECT_ID"
  }
}
```

### terraform.tfvars.example

```hcl
project_id = "my-gcp-project"
region     = "us-central1"

# Resource-specific variables below
```

Include relevant defaults for the chosen resource type with `environment = "<env>"` label.

### .gitignore

```
**/.terraform/*
*.tfstate
*.tfstate.*
*.tfvars
*.tfvars.json
!*.tfvars.example
override.tf
override.tf.json
*_override.tf
*_override.tf.json
*tfplan*
.terraformrc
terraform.rc
```

## Step 4 — Generate Module Files

Each module directory follows:

```
modules/<name>/
├── main.tf          # Resource definitions
├── variables.tf     # Module input variables
└── outputs.tf       # Module outputs
```

### Resource Templates

#### `gcs` — Google Cloud Storage Bucket

**variables.tf:**
```hcl
variable "project_id" { type = string }
variable "bucket_name" { type = string }
variable "location" { type = string; default = "US" }
variable "storage_class" { type = string; default = "STANDARD" }
variable "force_destroy" { type = bool; default = false }
variable "enable_versioning" { type = bool; default = true }
variable "uniform_bucket_level_access" { type = bool; default = true }
variable "labels" { type = map(string); default = {} }
variable "lifecycle_rules" {
  type = list(object({
    action    = object({ type = string, storage_class = optional(string) })
    condition = object({ age = optional(number), created_before = optional(string), with_state = optional(string) })
  }))
  default = []
}
```

**main.tf:**
```hcl
resource "google_storage_bucket" "this" {
  name                        = var.bucket_name
  project                     = var.project_id
  location                    = var.location
  storage_class               = var.storage_class
  force_destroy               = var.force_destroy
  uniform_bucket_level_access = var.uniform_bucket_level_access
  labels                      = var.labels

  versioning { enabled = var.enable_versioning }

  dynamic "lifecycle_rule" {
    for_each = var.lifecycle_rules
    content {
      action { type = lifecycle_rule.value.action.type; storage_class = lifecycle_rule.value.action.storage_class }
      condition { age = lifecycle_rule.value.condition.age; created_before = lifecycle_rule.value.condition.created_before; with_state = lifecycle_rule.value.condition.with_state }
    }
  }
}
```

**outputs.tf:**
```hcl
output "name" { value = google_storage_bucket.this.name }
output "url" { value = google_storage_bucket.this.url }
output "self_link" { value = google_storage_bucket.this.self_link }
```

#### `cloud-run` — Cloud Run Service

**variables.tf:**
```hcl
variable "project_id" { type = string }
variable "service_name" { type = string }
variable "region" { type = string }
variable "image" { type = string }
variable "port" { type = number; default = 8000 }
variable "cpu" { type = string; default = "1" }
variable "memory" { type = string; default = "512Mi" }
variable "min_instances" { type = number; default = 0 }
variable "max_instances" { type = number; default = 10 }
variable "env_vars" { type = map(string); default = {} }
variable "secret_env_vars" {
  type = map(object({ secret = string, version = string }))
  default = {}
}
variable "allow_unauthenticated" { type = bool; default = false }
variable "labels" { type = map(string); default = {} }
```

**main.tf:** Use `google_cloud_run_v2_service` with template, containers, scaling, env vars, and optional IAM binding for public access.

**outputs.tf:** Export `service_name`, `service_url`, `service_id`.

#### `cloud-sql` — Cloud SQL PostgreSQL

**variables.tf:**
```hcl
variable "project_id" { type = string }
variable "instance_name" { type = string }
variable "region" { type = string }
variable "database_version" { type = string; default = "POSTGRES_16" }
variable "tier" { type = string; default = "db-f1-micro" }
variable "disk_size" { type = number; default = 10 }
variable "disk_autoresize" { type = bool; default = true }
variable "availability_type" { type = string; default = "ZONAL" }
variable "database_name" { type = string }
variable "database_user" { type = string; default = "appuser" }
variable "authorized_networks" {
  type = list(object({ name = string, value = string }))
  default = []
}
variable "deletion_protection" { type = bool; default = true }
variable "labels" { type = map(string); default = {} }
```

**main.tf:** Use `google_sql_database_instance`, `google_sql_database`, `google_sql_user` with `random_password` for the database user password.

**outputs.tf:** Export `instance_name`, `instance_connection_name`, `database_name`, `database_user`, `database_password` (sensitive), `public_ip`.

#### `vpc` — VPC Network

**variables.tf:** `project_id`, `network_name`, `subnets` (list of objects with name, region, cidr), `enable_nat` (bool).

**main.tf:** Use `google_compute_network`, `google_compute_subnetwork` (dynamic), optional `google_compute_router` + `google_compute_router_nat`.

**outputs.tf:** Export `network_name`, `network_id`, `subnet_ids`.

#### `iam` — Service Account + IAM Bindings

**variables.tf:** `project_id`, `account_id`, `display_name`, `roles` (list of strings).

**main.tf:** Use `google_service_account` and `google_project_iam_member` for each role.

**outputs.tf:** Export `email`, `id`, `name`.

#### `full-stack` — Cloud Run + Cloud SQL + GCS

Generate all three modules (`cloud-run`, `cloud-sql`, `gcs`) and wire them together in `main.tf`:
- Cloud SQL instance
- GCS bucket for assets/uploads
- Cloud Run service with `DATABASE_URL` env var pointing to Cloud SQL via connection name
- IAM: service account for Cloud Run with `cloudsql.client` and `storage.objectViewer` roles

## Step 5 — Post-Generation

After writing all files:

1. List the files created
2. Show quick start:
   ```
   cd infra
   cp terraform.tfvars.example terraform.tfvars
   # Edit terraform.tfvars with your project details
   terraform init
   terraform plan
   terraform apply
   ```
3. Remind user to:
   - Set `project_id` in tfvars
   - Optionally enable remote state by copying `backend.tf.example` to `backend.tf`
   - Review the plan before applying
