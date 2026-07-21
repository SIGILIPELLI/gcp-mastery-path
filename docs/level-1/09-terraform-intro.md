# 09 · Infrastructure as Code (Terraform on GCP)

Every resource in this level so far was created with an imperative `gcloud`
command — run it once, and the resource exists, but there's no durable
record of *what should exist*. **Infrastructure as Code (IaC)** flips that:
you declare the desired state in a file, check it into version control, and
a tool reconciles reality to match it. GCP's own native option is
**Deployment Manager**, now effectively superseded in Google's own
recommendations by **Terraform**, the cross-cloud industry-standard tool —
this module focuses on Terraform.

## Install Terraform

```bash
brew install terraform          # macOS
# or download from https://developer.hashicorp.com/terraform/install

terraform version
```

## Provider setup

Terraform talks to GCP through the Google provider, authenticated using the
Application Default Credentials you set up in the first module of this
level:

```bash
gcloud auth application-default login
```

```hcl
# main.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = "gcp-mastery-path-123"
  region  = "us-central1"
}
```

## Declaring resources

```hcl
# main.tf (continued)
resource "google_storage_bucket" "assets" {
  name          = "gcp-mastery-path-123-tf-assets"
  location      = "US-CENTRAL1"
  force_destroy = true

  uniform_bucket_level_access = true
}

resource "google_compute_network" "app_vpc" {
  name                    = "app-vpc-tf"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "app_subnet" {
  name          = "app-subnet-tf"
  network       = google_compute_network.app_vpc.id
  region        = "us-central1"
  ip_cidr_range = "10.10.0.0/24"
}

resource "google_compute_firewall" "allow_internal" {
  name    = "app-vpc-tf-allow-internal"
  network = google_compute_network.app_vpc.id

  allow {
    protocol = "tcp"
  }
  allow {
    protocol = "udp"
  }
  source_ranges = ["10.10.0.0/24"]
  direction     = "INGRESS"
}
```

Notice the bucket, network, subnet, and firewall rule are declared together,
and `google_compute_subnetwork.app_subnet` **refers to**
`google_compute_network.app_vpc.id` — Terraform builds a dependency graph
from these references and creates resources in the right order
automatically.

## The core workflow

```bash
# Download the provider plugin declared in main.tf
terraform init

# Show what would change, without changing anything
terraform plan

# Apply those changes (prompts for confirmation)
terraform apply

# Destroy every resource this configuration manages
terraform destroy
```

`terraform plan` is the single most important habit to build: it always
tells you exactly what will be created, changed, or destroyed **before**
anything happens, using a `+`/`-`/`~` diff format:

```text
Terraform will perform the following actions:

  # google_storage_bucket.assets will be created
  + resource "google_storage_bucket" "assets" {
      + name     = "gcp-mastery-path-123-tf-assets"
      + location = "US-CENTRAL1"
      ...
    }

Plan: 4 to add, 0 to change, 0 to destroy.
```

## State

Terraform tracks what it created in a **state file** (`terraform.tfstate`
by default) — this is how `plan`/`apply`/`destroy` know what already exists
versus what's new. For solo learning, local state is fine. For anything
shared with a team, store state remotely instead (a GCS bucket is the
natural choice on GCP) so two people don't clobber each other's local state
file:

```hcl
terraform {
  backend "gcs" {
    bucket = "gcp-mastery-path-123-tfstate"
    prefix = "terraform/state"
  }
}
```

## Variables and outputs

```hcl
# variables.tf
variable "project_id" {
  type        = string
  description = "GCP project ID"
}

# main.tf
provider "google" {
  project = var.project_id
  region  = "us-central1"
}

# outputs.tf
output "bucket_url" {
  value = google_storage_bucket.assets.url
}
```

```bash
terraform apply -var="project_id=gcp-mastery-path-123"
terraform output bucket_url
```

## Cleanup

```bash
terraform destroy
```

Always prefer `terraform destroy` over manually deleting resources with
`gcloud` when Terraform manages them — manual deletion leaves stale entries
in the state file that cause confusing errors on the next `plan`.

## Cheat sheet

| Command | Purpose |
|---|---|
| `terraform init` | Download providers, set up the working directory. |
| `terraform plan` | Preview changes without applying them. |
| `terraform apply` | Create/update resources to match the config. |
| `terraform destroy` | Delete every resource the config manages. |
| `terraform output` | Print declared output values. |
| `terraform state list` | List resources tracked in the current state. |
| `resource "google_X" "name" { ... }` | Declare a GCP resource. |
| `var.name` / `variable "name" {}` | Parameterize a configuration. |

## Exercise

Write a Terraform configuration that declares a Cloud Storage bucket and a
custom VPC with one subnet (reusing the pattern above). Run `terraform plan`
and read the diff carefully before running `terraform apply`. Confirm the
resources exist via `gcloud storage buckets list` / `gcloud compute networks
list`, then run `terraform destroy` and confirm they're gone.
