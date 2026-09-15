---
description: "Cost Optimization at Scale — Level 1-3 mentioned individual cost levers in passing (Spot VMs, budgets). At fleet scale, cost optimization becomes a…"
---

# 06 · Cost Optimization at Scale

Level 1-3 mentioned individual cost levers in passing (Spot VMs, budgets).
At fleet scale, cost optimization becomes a discipline: committed-use
discounts, rightsizing at the recommender level, and cost attribution
across dozens of projects/teams. This module covers the mechanics —
qualitatively, without quoting specific dollar figures since actual
pricing varies by region, commitment term, and changes over time.

## Committed Use Discounts (CUDs) vs. Sustained Use Discounts

Compute Engine applies **sustained use discounts** automatically for
instances that run a large fraction of the billing month — no action
needed. **Committed use discounts** require an upfront 1- or 3-year
commitment to a specific amount of vCPU/memory (resource-based) or spend
(flexible/Spend-based CUDs), in exchange for a steeper discount than
sustained use alone.

```bash
gcloud compute commitments create prod-cud-1yr \
  --region=us-central1 \
  --plan=twelve-month \
  --resources=vcpu=64,memory=256GB
```

```bash
gcloud compute commitments list --region=us-central1
# NAME             REGION         STATUS  PLAN          START_TIME
# prod-cud-1yr     us-central1    ACTIVE  TWELVE_MONTH  2026-08-25
```

**Gotcha — a CUD commits you to paying for that capacity regardless of
actual usage.** If workloads shrink mid-commitment (successful cost
optimization elsewhere, or a service getting decommissioned), the
commitment still bills at the committed level — CUDs are a bet on
*minimum* sustained baseline usage, not a cap. Size commitments to your
demonstrated floor (e.g., last 90 days' minimum, not average) rather than
current peak usage.

## Rightsizing recommendations

```bash
gcloud recommender recommendations list \
  --project=my-project \
  --recommender=google.compute.instance.MachineTypeRecommender \
  --location=us-central1-a \
  --format="table(recommenderSubtype, primaryImpact.category, content.operationGroups)"
```

```bash
# RECOMMENDER_SUBTYPE     PRIMARY_IMPACT.CATEGORY
# CHANGE_MACHINE_TYPE     COST
```

Apply a recommendation via the referenced operation, or manually after
review:

```bash
gcloud compute instances stop web-01 --zone=us-central1-a
gcloud compute instances set-machine-type web-01 \
  --zone=us-central1-a --machine-type=e2-standard-4
gcloud compute instances start web-01 --zone=us-central1-a
```

**Gotcha — machine-type changes require a stop/start, meaning downtime**
for anything not behind a load balancer with other healthy backends.
Rightsizing a fleet of stateless instances behind an MIG is safe (rolling
replace); rightsizing a lone stateful VM needs a maintenance window.

## Idle and orphaned resource cleanup

The most common silent cost leak is resources nobody is using anymore:

```bash
# Unattached persistent disks
gcloud compute disks list --filter="-users:*" --format="table(name,zone,sizeGb)"

# Reserved IPs not attached to anything
gcloud compute addresses list --filter="status=RESERVED"

# Idle load balancer forwarding rules with no healthy backends
gcloud compute backend-services list --format="table(name,backends)"
```

```bash
gcloud recommender recommendations list \
  --project=my-project \
  --recommender=google.compute.disk.IdleResourceRecommender \
  --location=us-central1-a
```

**Gotcha — a stopped VM still bills for its persistent disk and any
reserved static IP.** "Stopped" only removes compute/vCPU billing; disks
and reserved (as opposed to ephemeral) external IPs keep accruing charges
until explicitly deleted or released — a common surprise after a large
stop-everything cost-cutting pass that didn't also clean up attached
storage/networking.

## Cost attribution with labels

```bash
gcloud compute instances create web-01 \
  --zone=us-central1-a \
  --labels=team=orders,env=prod,cost-center=eng-platform
```

```bash
bq query --use_legacy_sql=false '
SELECT
  labels.value AS team,
  SUM(cost) AS total_cost
FROM `billing_export.gcp_billing_export_v1_XXXXXX`,
UNNEST(labels) AS labels
WHERE labels.key = "team"
GROUP BY team
ORDER BY total_cost DESC'
```

Billing export to BigQuery (enabled once per billing account) plus
consistent labeling turns "why did the bill go up" from a manual dig
through the console into a repeatable SQL query — but only if labels are
enforced, not optional.

**Gotcha — labels aren't retroactive.** A resource created without a
`team` label shows up as unattributed in cost-by-team queries even after
you add the label later — labeling has to be enforced at creation (an org
policy or a Cloud Build/Terraform lint step rejecting unlabeled resources)
for the attribution query to actually be complete.

## Enforcing labels via org policy

```yaml
# require-labels-policy.yaml
name: organizations/123456789012/policies/compute.requireLabels  # illustrative — enforce via a custom constraint
spec:
  rules:
    - enforce: true
```

In practice, mandatory labeling is usually enforced via a **custom
constraint** (org policies don't natively cover "require label X" as a
built-in constraint) or a pre-commit/CI check on Terraform plans rejecting
resources missing required label keys — worth deciding explicitly which
mechanism your org uses rather than assuming org policy alone covers it.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud compute commitments create` | Lock in a CUD for a sustained usage floor. |
| `gcloud recommender recommendations list ...MachineTypeRecommender` | Get automated rightsizing suggestions. |
| `gcloud compute disks list --filter="-users:*"` | Find unattached (orphaned) persistent disks. |
| `gcloud compute addresses list --filter="status=RESERVED"` | Find reserved IPs not attached to a resource. |
| `--labels=team=,env=,cost-center=` | Enable per-team/per-env cost attribution via billing export. |
| Billing export to BigQuery | Turn cost analysis into a repeatable SQL query. |

## How It Actually Works

Cost optimization at fleet scale relies on the same rated-usage pipeline
covered in Level 2, but the levers that matter change once volume is
large enough for discount structures to dominate simple resource
right-sizing. Committed Use Discounts are a **forward-looking rate
commitment** — you commit to a baseline spend level and GCP applies a
discounted rate to usage up to that commitment automatically, with no
change to how resources actually run, which is why CUD planning is
fundamentally a usage-forecasting exercise rather than an engineering
one. BigQuery flat-rate/editions pricing flips the billing model
entirely: instead of paying per byte scanned, you reserve query
processing capacity (slots) at a fixed price, which only pays off once
your organization's aggregate on-demand query cost would exceed the
reservation cost — the crossover point is a real number computable from
historical billing export data, not a rule of thumb. Recommender's
machine-generated rightsizing suggestions work by analyzing actual
utilization telemetry (the same Cloud Monitoring time series from Level
1) against provisioned capacity, surfacing a specific alternate machine
type only when sustained utilization data supports it.

## 🔀 Related lessons on other tracks

- [AI/ML — 08 · Cost Optimization & Efficient Inference](https://sigilipelli.github.io/ai-ml-mastery-path/level-4/08-efficient-inference/)
- [AWS — Cost Optimization at Scale](https://sigilipelli.github.io/aws-mastery-path/level-4/06-cost-optimization-at-scale/)
- [Azure — 09 · Cost Management & Optimization](https://sigilipelli.github.io/azure-mastery-path/level-3/09-cost-management-optimization/)

## Exercise

Given a hypothetical fleet with a well-established usage floor, write the
`gcloud compute commitments create` command for a 1-year resource-based CUD
sized to that floor (not current peak) and explain your reasoning. Then
write a `bq` query against a billing export table that sums cost by `team`
label and separately identifies cost with no `team` label at all — describe
what enforcement mechanism you'd add so that second bucket shrinks to zero
going forward.
