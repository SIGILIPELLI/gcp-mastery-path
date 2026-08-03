# 09 · Cost Management & Billing

Every module in this course has said "remember to clean up" for a reason:
GCP bills for what exists, not what you're actively using. This module is
about the tools that stop a forgotten resource — or a genuine traffic spike —
from becoming a surprise invoice: budgets, alerts, billing export, and
labels that make a bill actually readable.

## The billing account hierarchy

```
Billing Account
 └── Project (gcp-mastery-path-123)
      └── Resources (VMs, buckets, clusters, ...)
```

A **billing account** is linked to one or more projects; every resource's
cost rolls up through its project to whichever billing account that project
is linked to. Checking the link is the first troubleshooting step when
"but I set a budget" turns out not to have applied to the resource in
question:

```bash
gcloud billing projects describe gcp-mastery-path-123
# billingAccountName: billingAccounts/012345-6789AB-CDEF01
# billingEnabled: true

gcloud billing accounts list
# ACCOUNT_ID            NAME              OPEN
# 012345-6789AB-CDEF01  My Billing Account  True
```

## Budgets and alert thresholds

A **budget** doesn't cap spend or stop anything automatically by default —
it sends notifications when spend crosses thresholds you define:

```bash
gcloud billing budgets create \
  --billing-account=012345-6789AB-CDEF01 \
  --display-name="Monthly Learning Budget" \
  --budget-amount=50USD \
  --threshold-rule=percent=0.5 \
  --threshold-rule=percent=0.9 \
  --threshold-rule=percent=1.0
```

```bash
gcloud billing budgets list --billing-account=012345-6789AB-CDEF01 \
  --format="table(displayName,amount.specifiedAmount.units)"
# DISPLAY_NAME              UNITS
# Monthly Learning Budget   50
```

At 50%, 90%, and 100% of $50 for the billing period, everyone with the
Billing Account Administrator/Viewer role gets an email — nothing is
throttled or shut off unless you separately wire a Pub/Sub notification to
automation that reacts to it (see below).

## Reacting to a budget alert automatically

Budgets can publish to a Pub/Sub topic on every threshold crossing, which a
Cloud Function can consume to take real action (e.g. disabling billing on a
runaway sandbox project — a drastic, last-resort move for real production
projects, but reasonable for a personal learning project):

```bash
gcloud pubsub topics create budget-alerts

gcloud billing budgets create \
  --billing-account=012345-6789AB-CDEF01 \
  --display-name="Auto-capped Sandbox Budget" \
  --budget-amount=20USD \
  --threshold-rule=percent=1.0 \
  --all-updates-rule-pubsub-topic=projects/gcp-mastery-path-123/topics/budget-alerts
```

A Cloud Function subscribed to `budget-alerts` receives a message with
`costAmount`, `budgetAmount`, and the project — from there it's ordinary
Pub/Sub-triggered code (Module 04) that calls `gcloud billing projects
link --billing-account=""` to disconnect billing (which also stops the
resources) if you choose to go that far.

## Cost breakdown by label

Untagged resources produce a bill that's just one big number. **Labels**
(key-value pairs on most GCP resources) let the billing export/reports break
spend down by team, environment, or feature:

```bash
gcloud compute instances create web-vm-1 \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --labels=env=staging,team=platform

gcloud compute instances list --format="table(name,labels.list())"
# NAME       LABELS
# web-vm-1   env=staging,team=platform
```

```bash
# Filter cost/inventory queries by label
gcloud compute instances list --filter="labels.env=staging"
```

## Billing export to BigQuery

For real analysis (spend trends, per-service breakdowns, anomaly detection),
export detailed billing data to BigQuery rather than reading the console UI
by hand:

```bash
bq mk --dataset gcp-mastery-path-123:billing_export

gcloud billing accounts get-iam-policy 012345-6789AB-CDEF01
```

Billing export itself is configured in the Cloud Console under Billing →
Billing export (there's no `gcloud` command to enable it — the destination
dataset must exist first, which is the `bq mk` step above). Once flowing:

```sql
-- Total spend by service, last 30 days
SELECT
  service.description AS service,
  SUM(cost) AS total_cost
FROM `gcp-mastery-path-123.billing_export.gcp_billing_export_v1_*`
WHERE _PARTITIONTIME > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY service
ORDER BY total_cost DESC;
```

```text
+------------------------+-------------+
| service                | total_cost  |
+------------------------+-------------+
| Compute Engine         | 34.21       |
| Kubernetes Engine      | 12.05       |
| Cloud Storage          | 1.14        |
+------------------------+-------------+
```

## Recommender: right-sizing and idle resources

GCP's Recommender service surfaces machine-generated cost suggestions based
on actual usage, without you having to hunt manually:

```bash
gcloud recommender recommendations list \
  --project=gcp-mastery-path-123 \
  --location=us-central1-a \
  --recommender=google.compute.instance.MachineTypeRecommender
```

```text
NAME                                   RECOMMENDER_SUBTYPE    STATE
projects/.../recommendations/abc-123   CHANGE_MACHINE_TYPE    ACTIVE
```

Common recommenders worth checking periodically: idle VMs
(`google.compute.instance.IdleResourceRecommender`), oversized persistent
disks, and unattached IP addresses still being billed.

## Committed use discounts (qualitative)

For steady, predictable long-running workloads, GCP offers **Committed Use
Discounts (CUDs)** — a discount (often 20-57% depending on term/resource) in
exchange for committing to 1 or 3 years of spend on a resource type/region.
This is a purely financial commitment decision (not something to provision
casually while learning) — mentioned here so the term is recognizable when
it shows up in a real production cost review, not as something to try in a
sandbox project.

## Gotchas

- **A budget alert is a notification, not a spending cap.** Nothing stops
  automatically at 100% unless you build the Pub/Sub → Cloud Function →
  billing-disconnect pipeline yourself.
- **Billing export only contains data from the point it was enabled
  forward** — there's no retroactive backfill of cost history from before
  you turned it on.
- **Labels must be added *before* they're useful for cost breakdown** —
  labeling a resource after the fact only affects cost attribution going
  forward, not past usage.
- **Recommender suggestions are based on historical usage patterns**, so a
  newly created resource won't have recommendations yet — give it time
  before treating an empty recommendation list as "already optimal."
- **Disabling billing on a project stops everything, hard** — every
  resource in it stops functioning immediately; treat the auto-disconnect
  pattern above as a sandbox-only safety net, never a production pattern.

## Cleanup

```bash
gcloud billing budgets delete <budget-id> --billing-account=012345-6789AB-CDEF01
gcloud compute instances delete web-vm-1 --zone=us-central1-a --quiet
bq rm -r -f gcp-mastery-path-123:billing_export
gcloud pubsub topics delete budget-alerts
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud billing projects describe` | Confirm which billing account a project is linked to. |
| `gcloud billing budgets create --threshold-rule=percent=` | Create a budget with alert thresholds. |
| `--all-updates-rule-pubsub-topic=` | Wire budget alerts to Pub/Sub for automation. |
| `--labels=key=value` (on most `create` commands) | Tag resources for cost attribution. |
| `bq mk --dataset` | Create the BigQuery dataset billing export writes into. |
| `gcloud recommender recommendations list` | Get machine-generated cost/right-sizing suggestions. |

## Exercise

Create a $10 budget with 50%/90%/100% thresholds on your billing account,
and confirm the alert email arrives after crossing a threshold (or simulate
it by lowering the amount below current spend). Label two or three existing
resources with `env=` and `team=`, then run a `list --filter="labels.env=..."`
query to confirm the filter works. If billing export is enabled, run the
BigQuery query above and identify your single most expensive service this
month.
