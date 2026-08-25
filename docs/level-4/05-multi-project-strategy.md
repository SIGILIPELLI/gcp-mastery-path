# 05 · Multi-Project Strategy (Organizations, Folders)

A single project works for a demo; a real org needs a resource hierarchy —
**Organization → Folders → Projects** — that maps to how teams, environments,
and billing actually need to be separated and governed.

## The hierarchy

```
Organization (example.com)
├── Folder: Production
│   ├── Project: orders-prod
│   └── Project: payments-prod
├── Folder: Non-Production
│   ├── Project: orders-staging
│   └── Project: orders-dev
└── Folder: Shared-Services
    ├── Project: net-host-project   (Shared VPC host)
    └── Project: logging-project    (centralized log sink)
```

```bash
gcloud resource-manager folders create \
  --display-name="Production" --organization=123456789012

gcloud projects create orders-prod \
  --folder=FOLDER_ID \
  --name="Orders Production"
```

```bash
gcloud projects list --filter="parent.id=FOLDER_ID"
# PROJECT_ID     NAME               PROJECT_NUMBER
# orders-prod    Orders Production  111111111111
```

**Why folders, not just flat projects with naming conventions:** IAM
bindings and org policies attached to a folder inherit to every project
under it automatically. "Production" folder gets stricter org policies
(no external IPs, mandatory CMEK) once, and every project created under it
inherits that from day one — no per-project checklist to forget.

## Project-per-environment vs. project-per-service

Two common axes to split projects along, often combined:

| Approach | Pros | Cons |
|---|---|---|
| One project per environment (dev/staging/prod), all services inside | Simple IAM surface | Blast radius: a bug in one service's IAM grant affects the whole environment |
| One project per service, per environment (`orders-prod`, `payments-prod`) | Tight blast radius, clean billing attribution per service | More projects to manage; cross-project networking (Shared VPC) becomes mandatory |

Most orgs beyond a handful of services land on project-per-service within
environment folders — the Shared VPC pattern (Level 3, Module 01) exists
specifically to make that split practical without every project needing
its own VPC and NAT gateway.

## Centralized logging and billing across projects

```bash
gcloud logging sinks create org-audit-sink \
  bigquery.googleapis.com/projects/logging-project/datasets/org_logs \
  --organization=123456789012 \
  --log-filter='logName:"cloudaudit.googleapis.com"' \
  --include-children
```

```bash
gcloud logging sinks describe org-audit-sink --organization=123456789012
# destination: bigquery.googleapis.com/projects/logging-project/datasets/org_logs
# includeChildren: true
# filter: logName:"cloudaudit.googleapis.com"
```

`--include-children` is what makes this org-wide rather than
single-project — every project under the org, present and future, has its
audit logs routed to one BigQuery dataset without per-project sink setup.

**Gotcha — sink service account needs write access at the destination,
granted separately.** Creating an org-level sink auto-generates a service
account, but it needs `roles/bigquery.dataEditor` on `logging-project`
explicitly — the sink silently drops logs (visible only via
`gcloud logging sinks describe` showing a permission error in exports, not
a loud failure) until that grant exists:

```bash
gcloud projects add-iam-policy-binding logging-project \
  --member="serviceAccount:$(gcloud logging sinks describe org-audit-sink --organization=123456789012 --format='value(writerIdentity)')" \
  --role="roles/bigquery.dataEditor"
```

## Billing: linking projects to accounts, and budget alerts per folder

```bash
gcloud billing projects link orders-prod --billing-account=012345-6789AB-CDEF01

gcloud billing budgets create \
  --billing-account=012345-6789AB-CDEF01 \
  --display-name="Production Folder Budget" \
  --budget-amount=... \
  --filter-projects=projects/orders-prod,projects/payments-prod \
  --threshold-rule=percent=0.9
```

**Gotcha — a project not linked to a billing account can't create billable
resources at all**, but it silently fails per-resource
(`FAILED_PRECONDITION: Billing account for project is not found`) rather
than blocking project creation itself — a project created via automation
without an explicit billing link looks fine until the first `gcloud
compute instances create` inside it.

## Cross-project references (Terraform)

```hcl
data "google_project" "orders_prod" {
  project_id = "orders-prod"
}

resource "google_project_iam_member" "shared_vpc_user" {
  project = "net-host-project"
  role    = "roles/compute.networkUser"
  member  = "serviceAccount:${data.google_project.orders_prod.number}-compute@developer.gserviceaccount.com"
}
```

Managing IAM bindings that span two projects from one Terraform root module
(rather than two separate applies with manually copy-pasted IDs) is what
keeps a multi-project hierarchy from becoming an untracked mess of
one-off `gcloud` commands nobody remembers running.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud resource-manager folders create` | Build the org's folder hierarchy. |
| `gcloud projects create --folder=` | Place a new project under a specific folder. |
| `gcloud logging sinks create --include-children` | Centralize logs org-wide into one destination. |
| `gcloud billing projects link` | Attach a project to a billing account (required for billable resources). |
| `gcloud billing budgets create --filter-projects=` | Budget alert scoped to a folder's set of projects. |

## Exercise

Design a three-folder hierarchy (Production, Non-Production,
Shared-Services) with at least two projects under Production. Write the
exact `gcloud resource-manager folders create` and `gcloud projects
create --folder=` commands, then write an org-level logging sink with
`--include-children` plus the IAM binding its writer identity needs at the
destination project. State which project-splitting approach
(per-environment vs. per-service) you'd choose for a 6-service system and
why.
