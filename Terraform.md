# Terraform + GCP — Production Guide

> Covers: **what Terraform is → setup → core workflow → deploying 3 microservices on GCP → remote state → production best practices → what to say in an interview**.

---

## 1. What is Terraform & Why

**Terraform** is an **Infrastructure as Code (IaC)** tool by HashiCorp. You write code (`.tf` files) that describes what cloud resources you want, and Terraform creates/updates/destroys them to match.

**Why not click-and-point in the GCP console?**
- Manual changes can't be reviewed, versioned, or reproduced.
- Teams can't collaborate on infrastructure the same way they do on code.
- Terraform gives you: **version control + peer review + automated drift detection**.

**How Terraform thinks:**
```
Your .tf files  →  Terraform Plan  →  Terraform Apply  →  Real GCP Resources
  (desired state)    (diff/preview)     (execution)
```

Terraform tracks what it created in a **state file** (`terraform.tfstate`). On the next run, it compares desired vs actual and only changes what's different.

---

## 2. Setup

### Install Terraform

```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Verify
terraform version
```

### Install & Configure gcloud CLI

```bash
# Install gcloud SDK (macOS)
brew install --cask google-cloud-sdk

# Authenticate
gcloud auth login
gcloud auth application-default login   # lets Terraform use your credentials

# Set your project
gcloud config set project YOUR_PROJECT_ID
```

### Enable required GCP APIs

```bash
gcloud services enable \
  run.googleapis.com \
  container.googleapis.com \
  artifactregistry.googleapis.com \
  storage.googleapis.com \
  cloudresourcemanager.googleapis.com \
  iam.googleapis.com
```

---

## 3. Core Concepts (Quick Mental Model)

| Concept | What it is | Analogy |
|---|---|---|
| **Provider** | Plugin to talk to a cloud (GCP, AWS) | Driver for your cloud |
| **Resource** | A single cloud thing (VM, bucket, service) | One LEGO brick |
| **Variable** | Input to your config (reusable, parameterised) | Function argument |
| **Output** | Value Terraform exposes after apply | Return value |
| **State** | Terraform's memory of what it created | Ledger / source of truth |
| **Module** | A reusable group of resources | A function / library |
| **Workspace** | Named state isolation (dev/staging/prod) | Git branch equivalent |

### The Core Workflow

```
terraform init     # download provider plugins, set up backend
terraform plan     # preview what will change (safe — no changes made)
terraform apply    # make the changes (prompts for confirmation)
terraform destroy  # tear everything down
```

Always run `plan` before `apply`. Treat the plan output like a PR diff — review before merging.

---

## 4. Project Structure

A clean, production-ready folder layout for 3 microservices:

```
infra/
├── main.tf               ← root module: ties everything together
├── variables.tf          ← input variable declarations
├── outputs.tf            ← output values
├── versions.tf           ← provider + terraform version locks
├── backend.tf            ← remote state config (GCS)
│
├── modules/
│   ├── cloud-run/        ← reusable Cloud Run module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── iam/              ← reusable IAM module
│       ├── main.tf
│       └── variables.tf
│
└── environments/
    ├── dev/
    │   ├── main.tf       ← calls root module with dev vars
    │   └── terraform.tfvars
    ├── staging/
    │   └── terraform.tfvars
    └── prod/
        └── terraform.tfvars
```

**Rule of thumb:** Put reusable infrastructure logic in `modules/`. Put environment-specific values in `environments/`.

---

## 5. The 3 Microservices We'll Deploy

We'll deploy three Cloud Run services:

| Service | Role | Port |
|---|---|---|
| `api-gateway` | Public-facing entry point, routes traffic | 8080 |
| `auth-service` | Handles login, JWT issuance | 8081 |
| `order-service` | Business logic, talks to auth | 8082 |

**Why Cloud Run?** Serverless containers — no VMs to manage, scales to zero, pay per request. Ideal for microservices.

---

## 6. Writing the Terraform Code

### `versions.tf` — Lock your versions (always do this)

```hcl
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"   # ~> means: 5.x only, not 6.x
    }
  }
}
```

> **Why lock versions?** Terraform or provider upgrades can introduce breaking changes. Locking versions makes your infrastructure reproducible across machines and CI pipelines.

---

### `variables.tf` — Declare all inputs

```hcl
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "GCP region to deploy into"
  type        = string
  default     = "us-central1"
}

variable "environment" {
  description = "Deployment environment: dev, staging, prod"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod"
  }
}

variable "services" {
  description = "Map of microservice names to their container images"
  type        = map(string)
  default = {
    api-gateway   = "gcr.io/YOUR_PROJECT/api-gateway:latest"
    auth-service  = "gcr.io/YOUR_PROJECT/auth-service:latest"
    order-service = "gcr.io/YOUR_PROJECT/order-service:latest"
  }
}
```

---

### `main.tf` — Deploy the 3 services

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
}

# --- Artifact Registry (where your container images live) ---
resource "google_artifact_registry_repository" "services" {
  location      = var.region
  repository_id = "microservices-${var.environment}"
  format        = "DOCKER"
  description   = "Container images for ${var.environment}"
}

# --- Service Account for Cloud Run services ---
resource "google_service_account" "cloud_run_sa" {
  account_id   = "cloud-run-sa-${var.environment}"
  display_name = "Cloud Run Service Account (${var.environment})"
}

# --- Deploy all 3 services using a loop ---
resource "google_cloud_run_v2_service" "microservices" {
  for_each = var.services   # creates one resource per service

  name     = "${each.key}-${var.environment}"
  location = var.region

  template {
    service_account = google_service_account.cloud_run_sa.email

    containers {
      image = each.value

      resources {
        limits = {
          cpu    = "1"
          memory = "512Mi"
        }
      }

      env {
        name  = "ENVIRONMENT"
        value = var.environment
      }
    }

    scaling {
      min_instance_count = var.environment == "prod" ? 1 : 0
      max_instance_count = var.environment == "prod" ? 10 : 3
    }
  }

  traffic {
    type    = "TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST"
    percent = 100
  }
}

# --- Make api-gateway publicly accessible ---
resource "google_cloud_run_v2_service_iam_member" "public_access" {
  name     = google_cloud_run_v2_service.microservices["api-gateway"].name
  location = var.region
  role     = "roles/run.invoker"
  member   = "allUsers"
}

# --- auth-service and order-service: only accessible by the SA ---
resource "google_cloud_run_v2_service_iam_member" "internal_access" {
  for_each = toset(["auth-service", "order-service"])

  name     = google_cloud_run_v2_service.microservices[each.key].name
  location = var.region
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.cloud_run_sa.email}"
}
```

---

### `outputs.tf` — Expose useful values

```hcl
output "service_urls" {
  description = "Public URLs for all deployed services"
  value = {
    for name, service in google_cloud_run_v2_service.microservices :
    name => service.uri
  }
}

output "api_gateway_url" {
  description = "Entry point URL (share this one)"
  value       = google_cloud_run_v2_service.microservices["api-gateway"].uri
}
```

After `apply`, run `terraform output` to see the live URLs.

---

## 7. Remote State in GCS (Critical for Teams)

By default, Terraform saves state locally (`terraform.tfstate`). This is fine alone — but in a team or CI pipeline, everyone needs to share the same state. Remote state solves this.

### Why remote state?
- **Shared truth:** Everyone sees the same infrastructure state.
- **State locking:** GCS + lock prevents two people applying at the same time (which would corrupt state).
- **Not in git:** State can contain secrets. Never commit it.

### Step 1 — Create the GCS bucket (do this once, manually or with a bootstrap script)

```bash
gsutil mb -l us-central1 gs://YOUR_PROJECT-terraform-state
gsutil versioning set on gs://YOUR_PROJECT-terraform-state   # recover from accidents
```

Enable versioning so you can roll back to a previous state if something goes wrong.

### Step 2 — `backend.tf`

```hcl
terraform {
  backend "gcs" {
    bucket = "YOUR_PROJECT-terraform-state"
    prefix = "microservices/prod"   # different prefix per environment
  }
}
```

Each environment gets its own path in the bucket:
```
gs://my-project-terraform-state/
  microservices/dev/terraform.tfstate
  microservices/staging/terraform.tfstate
  microservices/prod/terraform.tfstate
```

### Step 3 — Re-initialise with the remote backend

```bash
terraform init   # Terraform will detect the backend change and migrate state
```

### Saving and sharing the plan file

In CI pipelines, you save the plan so `apply` uses *exactly* what was reviewed:

```bash
# CI Step 1: Plan and save to file
terraform plan -out=tfplan.binary

# Optionally save human-readable plan for review
terraform show -json tfplan.binary | jq > tfplan.json

# CI Step 2: Apply the saved plan (no re-evaluation, no prompt)
terraform apply tfplan.binary
```

> **Why this matters:** Without `-out`, a second `apply` re-evaluates the plan — conditions may have changed between plan and apply. Saving the plan ensures you apply exactly what was reviewed.

---

## 8. Environment-Specific Variables

`environments/prod/terraform.tfvars`:
```hcl
project_id  = "my-company-prod"
region      = "us-central1"
environment = "prod"

services = {
  api-gateway   = "us-central1-docker.pkg.dev/my-company-prod/microservices-prod/api-gateway:v2.1.0"
  auth-service  = "us-central1-docker.pkg.dev/my-company-prod/microservices-prod/auth-service:v1.4.2"
  order-service = "us-central1-docker.pkg.dev/my-company-prod/microservices-prod/order-service:v3.0.1"
}
```

Run with:
```bash
terraform apply -var-file="environments/prod/terraform.tfvars"
```

**Never use `latest` tag in prod.** Pin to an exact version tag. `latest` is unpredictable — a new image push silently changes what gets deployed.

---

## 9. Production Best Practices

### Code Quality

| Practice | Why |
|---|---|
| **Lock provider versions** (`~> 5.0`) | Prevent surprise breaking changes |
| **Use `terraform fmt`** | Consistent formatting, no style debates |
| **Use `terraform validate`** | Catch syntax errors before plan |
| **Use `tflint`** | Catch logic errors, deprecated arguments |
| **Use `checkov` or `tfsec`** | Security scanning (detects open firewall rules, public buckets) |

```bash
terraform fmt -recursive    # format all .tf files
terraform validate          # check syntax + references
tflint                      # lint rules
checkov -d .                # security scan
```

---

### State Safety

```
✅ Remote state (GCS) with versioning
✅ State locking (GCS does this automatically)
✅ Never commit .tfstate to git  →  add to .gitignore
✅ Restrict who can access the state bucket (IAM)
✅ Save plan with -out in CI before apply
```

`.gitignore` for Terraform:
```gitignore
.terraform/
.terraform.lock.hcl      # commit this one — it locks provider versions
*.tfstate
*.tfstate.backup
tfplan.binary
*.tfvars                 # if they contain secrets
```

Wait — `.terraform.lock.hcl` **should** be committed. It pins exact provider checksums so every machine gets the same provider binary.

---

### Secrets — Never hardcode them

```hcl
# ❌ WRONG — never do this
resource "google_cloud_run_v2_service" "bad" {
  template {
    containers {
      env {
        name  = "DB_PASSWORD"
        value = "supersecret123"   # this ends up in state AND git
      }
    }
  }
}

# ✅ RIGHT — reference from Secret Manager
data "google_secret_manager_secret_version" "db_password" {
  secret = "db-password-${var.environment}"
}

resource "google_cloud_run_v2_service" "good" {
  template {
    containers {
      env {
        name = "DB_PASSWORD"
        value_source {
          secret_key_ref {
            secret  = data.google_secret_manager_secret_version.db_password.secret
            version = "latest"
          }
        }
      }
    }
  }
}
```

---

### CI/CD Pipeline (GitHub Actions example)

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:

jobs:
  terraform:
    runs-on: ubuntu-latest
    permissions:
      id-token: write    # for Workload Identity Federation (keyless auth)
      contents: read

    steps:
      - uses: actions/checkout@v4

      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
          service_account: ${{ secrets.TF_SERVICE_ACCOUNT }}

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.5"

      - name: Init
        run: terraform init

      - name: Format check
        run: terraform fmt -check -recursive

      - name: Validate
        run: terraform validate

      - name: Plan
        run: terraform plan -out=tfplan.binary
        env:
          TF_VAR_project_id: ${{ secrets.GCP_PROJECT_ID }}

      # Only apply on push to main — not on PRs
      - name: Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan.binary
```

**Workload Identity Federation** = keyless auth between GitHub Actions and GCP. No service account key JSON file stored as a secret. Highly recommended.

---

### Drift Detection

**Drift** = when someone manually changes something in the GCP console, making reality differ from your Terraform state.

```bash
terraform plan   # shows drift as unexpected changes
terraform refresh # updates state to match reality (careful — doesn't revert)
```

In production, run `terraform plan` on a schedule (nightly) and alert if there's unexpected drift. This catches manual changes before they cause incidents.

---

### Modules — DRY up repeated patterns

Instead of copy-pasting Cloud Run config three times, put it in a module:

`modules/cloud-run/main.tf`:
```hcl
variable "name"            { type = string }
variable "image"           { type = string }
variable "region"          { type = string }
variable "service_account" { type = string }
variable "is_public"       { type = bool; default = false }

resource "google_cloud_run_v2_service" "this" {
  name     = var.name
  location = var.region
  # ... full config here
}

output "url" { value = google_cloud_run_v2_service.this.uri }
```

Call it from `main.tf`:
```hcl
module "api_gateway" {
  source         = "./modules/cloud-run"
  name           = "api-gateway-prod"
  image          = "...api-gateway:v2.1.0"
  region         = var.region
  service_account = google_service_account.cloud_run_sa.email
  is_public      = true
}
```

---

## 10. Common Commands Reference

```bash
# Initialise (run after cloning or changing backend/providers)
terraform init

# See what will change — always run this first
terraform plan

# Apply with a saved plan (CI use)
terraform plan -out=tfplan.binary
terraform apply tfplan.binary

# Apply interactively (local dev)
terraform apply

# Destroy a specific resource without destroying everything
terraform destroy -target=google_cloud_run_v2_service.microservices[\"auth-service\"]

# Show current state
terraform show

# List all resources Terraform knows about
terraform state list

# Move a resource in state (renaming without recreation)
terraform state mv old_resource_name new_resource_name

# Import an existing GCP resource into state
terraform import google_cloud_run_v2_service.existing projects/PROJECT/locations/REGION/services/NAME

# See outputs
terraform output
terraform output api_gateway_url
```

---

## 11. GCP IAM — Least Privilege for Terraform

The service account running Terraform should have only what it needs:

```hcl
resource "google_project_iam_member" "terraform_sa_roles" {
  for_each = toset([
    "roles/run.admin",
    "roles/iam.serviceAccountUser",
    "roles/artifactregistry.admin",
    "roles/storage.objectAdmin",    # for GCS state bucket
  ])

  project = var.project_id
  role    = each.key
  member  = "serviceAccount:terraform@${var.project_id}.iam.gserviceaccount.com"
}
```

Never run Terraform with `roles/owner`. Grant the minimum set of roles required.

---

## Quick Reference

| Need | Command / Concept |
|---|---|
| Set up a project | `terraform init` with a `backend.tf` pointing to GCS |
| Preview changes safely | `terraform plan` |
| Save plan for CI | `terraform plan -out=tfplan.binary` |
| Deploy 3 services at once | `for_each` over a map of services |
| Separate dev/prod | Different `.tfvars` files + different GCS prefixes |
| Reuse patterns | Modules in `modules/` |
| No secrets in code | Google Secret Manager + `secret_key_ref` |
| Keyless CI auth | Workload Identity Federation (no JSON key) |
| Catch security issues | `checkov -d .` or `tfsec` |
| Handle drift | Scheduled `terraform plan` + alerting |

---

## One-liners to Memorise

- **Terraform** = declare desired state in `.tf` files → plan shows the diff → apply makes it real.
- **State** = Terraform's ledger of what it created; keep it remote in GCS, never in git.
- **Remote state** = GCS bucket with versioning + locking; one prefix per environment.
- **`-out` flag** = save the plan binary so `apply` uses exactly what was reviewed — no surprises.
- **`for_each`** = deploy N resources from a map without copy-paste; one block deploys all 3 services.
- **Modules** = reusable resource groups; keeps environments DRY.
- **Pin versions** = `required_version` + `version = "~> 5.0"` → reproducible, no surprise breaks.
- **Secrets** = never in `.tf` files or `.tfvars` → use Secret Manager; Terraform references, not stores.
- **Least privilege** = Terraform SA gets only the roles it needs — not `roles/owner`.
- **Workload Identity** = keyless GitHub Actions → GCP auth; no service account key file needed.
- **Drift** = manual console changes diverge from state; catch with scheduled `terraform plan`.
- **`terraform fmt` + `validate` + `tflint` + `checkov`** = your pre-commit quality gate.
