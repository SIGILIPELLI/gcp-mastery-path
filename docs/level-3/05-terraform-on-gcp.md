# 05 · Terraform on GCP (Advanced)

Level 1's Terraform intro covered a single resource block or two. This
module covers the patterns that matter once a codebase has more than one
environment: **modules**, **remote state with locking**, **workspaces**,
and **import** for adopting resources that already exist.

## Remote state with locking

Local state (`terraform.tfstate` on disk) doesn't survive a team. Store it
in a GCS bucket, which Terraform also uses for locking so two people can't
apply concurrently.

```bash
gsutil mb -l us-central1 gs://my-project-tfstate
gsutil versioning set on gs://my-project-tfstate
```

```hcl
# backend.tf
terraform {
  backend "gcs" {
    bucket = "my-project-tfstate"
    prefix = "env/prod"
  }
}
```

```bash
terraform init
# Initializing the backend...
# Successfully configured the backend "gcs"!
```

Versioning on the bucket means a corrupted or bad-apply state file can be
rolled back to a prior generation with `gsutil cp
gs://my-project-tfstate/env/prod/default.tfstate#GENERATION ...` — treat
this as your state's undo history.

**Gotcha — the GCS backend locks via an object, not a distributed lock
service.** If `terraform apply` is killed mid-run (Ctrl-C doesn't always
clean up, a CI job gets OOM-killed), the lock object can be left behind,
and the next run fails with `Error acquiring the state lock`. Recovery is
`terraform force-unlock LOCK_ID` — but only after confirming no other apply
is actually still running, since forcing an unlock during a live apply can
corrupt state.

## Modules

A module is a reusable, parameterized bundle of resources.

```hcl
# modules/cloud-run-service/main.tf
variable "service_name" { type = string }
variable "image"        { type = string }
variable "region"       { type = string, default = "us-central1" }

resource "google_cloud_run_v2_service" "this" {
  name     = var.service_name
  location = var.region
  template {
    containers {
      image = var.image
    }
  }
}

output "url" {
  value = google_cloud_run_v2_service.this.uri
}
```

```hcl
# environments/prod/main.tf
module "orders_api" {
  source       = "../../modules/cloud-run-service"
  service_name = "orders-api"
  image        = "us-central1-docker.pkg.dev/my-project/app/orders:v3"
  region       = "us-central1"
}

output "orders_api_url" {
  value = module.orders_api.url
}
```

```bash
terraform plan
# Plan: 1 to add, 0 to change, 0 to destroy.
terraform apply -auto-approve
# module.orders_api.google_cloud_run_v2_service.this: Creating...
# Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
# Outputs:
# orders_api_url = "https://orders-api-abc123-uc.a.run.app"
```

**Gotcha — module source changes require re-init.** Editing `source` (e.g.
switching a module from a local path to a Git ref) doesn't take effect
until `terraform init -upgrade`; a plain `terraform plan` will silently
keep using the previously initialized module copy.

## Workspaces for environment separation

```bash
terraform workspace new staging
terraform workspace new prod
terraform workspace select staging
terraform workspace show
# staging
```

```hcl
resource "google_cloud_run_v2_service" "this" {
  name     = "orders-api-${terraform.workspace}"
  location = var.region
  ...
}
```

Each workspace gets its own state file within the same backend
(`env/prod` prefix effectively becomes per-workspace under the hood), so
`staging` and `prod` never collide even sharing one `.tf` codebase.

**Gotcha — workspaces are not a substitute for separate state buckets in
regulated environments.** Because all workspaces share one backend
configuration, anyone with read access to that GCS bucket can see every
environment's state, including prod secrets that ended up in state (e.g.
generated passwords). For strict separation, use distinct backend
configs/buckets per environment instead of workspaces, and treat this
module's workspace example as fine for dev/staging convenience only.

## Importing existing resources

Adopting a resource that was created manually (`gcloud` or console) into
Terraform management:

```bash
terraform import google_storage_bucket.assets my-project/assets-bucket
```

```bash
terraform plan
# google_storage_bucket.assets: Refreshing state...
# No changes. Your infrastructure matches the configuration.
```

If `plan` shows changes immediately after an import, your `.tf` resource
block doesn't yet match the real resource's settings — write the block to
mirror the actual config (checked via `gcloud storage buckets describe`)
*before* importing, or the next apply will "fix" a resource that wasn't
actually broken.

**Gotcha — `import` only imports the resource, not dependent references.**
IAM bindings, lifecycle rules, and notification configs attached to that
bucket need their own separate `import` commands; a single `terraform
import` doesn't recursively pull in everything GCP considers "related" to
the resource.

## Cheat sheet

| Command | Purpose |
|---|---|
| `backend "gcs" {}` | Store state remotely with locking. |
| `terraform force-unlock` | Recover from a stuck state lock (verify no live apply first). |
| `module "x" { source = ... }` | Reuse a parameterized resource bundle. |
| `terraform init -upgrade` | Pick up a changed module source. |
| `terraform workspace new/select` | Separate state per environment within one backend. |
| `terraform import RESOURCE ID` | Bring an existing GCP resource under Terraform management. |

## Exercise

Create a GCS backend bucket with versioning, wire a `backend "gcs"` block
to it, and write a `cloud-run-service` module with `service_name`, `image`,
and `region` variables. Instantiate it twice (two different service names)
in the same root module, `terraform apply`, then delete one Cloud Run
service manually with `gcloud run services delete` and run `terraform plan`
to observe Terraform detecting and offering to recreate the drifted
resource.
