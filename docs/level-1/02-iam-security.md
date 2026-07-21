# 02 · IAM & Security Basics

Identity and Access Management (IAM) controls **who** (identity) can do
**what** (role/permissions) on **which resource**. Every action you've taken
so far — creating a project, enabling an API — was already authorized by
IAM behind the scenes. This module makes that explicit so you can grant
access safely instead of by trial and error.

## The three pieces of every IAM policy

An IAM binding is always: **member + role + resource**.

- **Member** — a Google account, service account, Google group, or
  Workspace/Cloud Identity domain. Members are identified by an email-like
  string, e.g. `user:you@example.com` or
  `serviceAccount:my-sa@project.iam.gserviceaccount.com`.
- **Role** — a named bundle of permissions, e.g. `roles/storage.objectViewer`
  bundles `storage.objects.get` and `storage.objects.list`.
- **Resource** — the project, folder, organization, or individual resource
  (like one bucket) the binding applies to. IAM policies are inherited
  downward: a role granted at the project level applies to every resource
  inside it.

```bash
# Grant a user the "Viewer" role on the whole project
gcloud projects add-iam-policy-binding gcp-mastery-path-123 \
  --member="user:teammate@example.com" \
  --role="roles/viewer"

# See the current policy
gcloud projects get-iam-policy gcp-mastery-path-123
```

## Role types

| Type | Example | Notes |
|---|---|---|
| **Basic** | `roles/owner`, `roles/editor`, `roles/viewer` | Broad, project-wide. Convenient but coarse — avoid in anything beyond a personal sandbox. |
| **Predefined** | `roles/storage.objectViewer`, `roles/compute.instanceAdmin.v1` | Scoped to one service, curated by Google. The default choice for real work. |
| **Custom** | `roles/myCustomDeployer` | You define the exact permission list. Use when predefined roles are too broad or too narrow. |

List roles and inspect what a role actually grants:

```bash
gcloud iam roles list --filter="name:roles/storage"

gcloud iam roles describe roles/storage.objectViewer
# includedPermissions:
# - storage.objects.get
# - storage.objects.list
```

## Principle of least privilege

Grant the **narrowest role, on the narrowest resource, for the shortest time**
that gets the job done. In practice:

- Prefer a predefined role scoped to one service over `roles/editor`.
- Prefer binding at the resource level (one bucket, one instance) over the
  project level when the access is genuinely that narrow.
- Prefer granting to a **group** over individual users when more than one or
  two people need the same access — you manage membership once instead of
  editing IAM policy per person.
- Review who has `roles/owner` periodically — it includes the ability to
  change IAM policy itself, so it should be held by very few identities.

```bash
# Narrow: viewer on just one bucket, not the whole project
gsutil iam ch user:teammate@example.com:objectViewer \
  gs://gcp-mastery-path-123-assets
```

## Service accounts

A **service account** is an identity for workloads, not humans — your Compute
Engine VM, Cloud Function, or a CI pipeline authenticates as one instead of
as a person's Google login. Every project gets a default Compute Engine
service account, but creating dedicated, narrowly-scoped ones per workload is
the recommended practice.

```bash
# Create a dedicated service account for an app
gcloud iam service-accounts create app-runtime \
  --display-name="App Runtime Service Account"

# Grant it only what it needs
gcloud projects add-iam-policy-binding gcp-mastery-path-123 \
  --member="serviceAccount:app-runtime@gcp-mastery-path-123.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Generate a key file for local/offsite use (avoid this when a
# metadata-server-based identity is available, e.g. on Compute Engine, Cloud
# Run, or GKE workload identity -- keys are a standing secret to manage)
gcloud iam service-accounts keys create key.json \
  --iam-account=app-runtime@gcp-mastery-path-123.iam.gserviceaccount.com
```

!!! warning "Service account keys are long-lived secrets"
    A downloaded JSON key works until you explicitly revoke it — there's no
    built-in expiry. Prefer attaching a service account directly to a
    Compute Engine VM, Cloud Run service, or Cloud Function (no key file
    needed at all) whenever the workload runs on GCP itself. Only create key
    files for genuinely external systems, and rotate/delete them when done:
    `gcloud iam service-accounts keys delete <KEY_ID> --iam-account=...`.

## Checking "who can do what"

Two directions matter: what can this identity do, and who can do this thing.

```bash
# What roles does a member have, project-wide?
gcloud projects get-iam-policy gcp-mastery-path-123 \
  --flatten="bindings[].members" \
  --filter="bindings.members:teammate@example.com" \
  --format="table(bindings.role)"

# Would a specific member be allowed a specific permission? (Policy Simulator
# in the console gives a fuller picture; this checks granted roles directly.)
gcloud iam roles describe roles/storage.objectViewer --format="value(includedPermissions)"
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud projects add-iam-policy-binding` | Grant a role to a member on a project. |
| `gcloud projects remove-iam-policy-binding` | Revoke a role from a member. |
| `gcloud projects get-iam-policy` | View the current IAM policy. |
| `gcloud iam roles list` | List available (predefined/custom) roles. |
| `gcloud iam roles describe <role>` | See exactly what permissions a role grants. |
| `gcloud iam service-accounts create` | Create a new service account. |
| `gcloud iam service-accounts keys create` | Generate a key file for a service account (use sparingly). |
| `gcloud iam service-accounts keys delete` | Revoke a service account key. |

## Exercise

Create a service account named `reporting-bot`. Grant it
`roles/storage.objectViewer` scoped to a single bucket you own (not the
whole project) using `gsutil iam ch`. Then run
`gcloud projects get-iam-policy` and confirm `reporting-bot` does **not**
appear with any project-wide role — only the bucket-level grant should
exist. Finally, list and then delete any key you created for it.
