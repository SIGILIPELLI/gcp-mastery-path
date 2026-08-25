# 01 · GCP Architecture Framework

Google publishes the **Architecture Framework** — five pillars for
evaluating any GCP design: Operational Excellence, Security/Privacy/
Compliance, Reliability, Cost Optimization, and Performance Optimization.
This module isn't new CLI syntax; it's the lens for reviewing everything
built in Levels 1-3, plus the tooling GCP gives you to audit against it.

## The five pillars, applied

| Pillar | Question it asks | GCP tooling |
|---|---|---|
| Operational Excellence | Can we deploy, observe, and recover without heroics? | Cloud Build, Cloud Monitoring, Cloud Logging, runbooks |
| Security, Privacy, Compliance | Least privilege? Data protected at rest/in transit/in use? | IAM, VPC-SC, SCC, CMEK, Assured Workloads |
| Reliability | Does it survive a zone/region failure? What's the RTO/RPO? | Multi-region, health checks, SLOs, chaos testing |
| Cost Optimization | Are we paying for capacity we use? | Committed use discounts, rightsizing recommender, budgets |
| Performance Optimization | Does it scale to 10x load without a redesign? | Load testing, autoscaling, caching, CDN |

Every design decision trades between these — a single-region deployment
optimizes cost and simplicity at the expense of reliability; that's a valid
choice *if stated explicitly*, not a default made by omission.

## Using the Architecture Review checklist in practice

Google's `gcloud recommender` surfaces automated suggestions mapped to
several pillars directly from your actual usage:

```bash
gcloud recommender recommendations list \
  --project=my-project \
  --recommender=google.compute.instance.MachineTypeRecommender \
  --location=us-central1-a
```

```bash
# NAME                                          PRIMARY_IMPACT
# .../recommendations/abcd-1234                 COST: reduce cost by resizing e2-standard-8 to e2-standard-4
```

```bash
gcloud recommender recommendations list \
  --project=my-project \
  --recommender=google.iam.policy.Recommender \
  --location=global
# .../recommendations/efgh-5678   SECURITY: remove unused role roles/editor from user:dev@example.com
```

Recommenders don't auto-apply — each recommendation includes an
`associatedInsights` reference and an operation you review and apply
explicitly:

```bash
gcloud recommender recommendations mark-claimed \
  --recommendation=abcd-1234 \
  --recommender=google.compute.instance.MachineTypeRecommender \
  --location=us-central1-a \
  --etag=ETAG_VALUE
```

**Gotcha — recommenders need history to generate signal.** A newly created
project or resource often shows zero recommendations for weeks — the
underused-resource and IAM recommenders both need a baseline observation
window (typically ~28 days) before they have enough data to recommend
anything. Don't read "no recommendations" as "this is optimal" for a young
project.

## A worked trade-off: the same service, two ways

**Design A — cost-optimized:** single-region Cloud Run, Cloud SQL with no
replica, no CDN. Cheapest, RTO measured in hours (restore from backup), no
protection from a regional outage.

**Design B — reliability-optimized:** two-region Cloud Run behind a global
LB, Spanner multi-region, Cloud CDN. RTO near-zero, but Spanner's
per-node cost and cross-region write latency are the price, and the design
is meaningfully harder to operate (more moving parts to monitor).

Neither is "correct" in isolation — Design A is right for an internal tool
where a few hours of downtime is a non-event; Design B is right for a
customer-facing payments path. The Architecture Framework's value is
forcing that justification to be explicit and reviewed, rather than
inherited from whatever the last engineer happened to build.

## Landing zone concept

A **landing zone** is the pre-built org structure — folders, projects,
baseline IAM, baseline org policies, logging sinks — that new workloads
land into, so every new project starts compliant instead of getting
audited into compliance after the fact.

```bash
gcloud resource-manager folders create --display-name="Production" --organization=123456789012
gcloud resource-manager folders create --display-name="Non-Production" --organization=123456789012
gcloud resource-manager folders create --display-name="Shared-Services" --organization=123456789012
```

```bash
gcloud resource-manager folders list --organization=123456789012
# DISPLAY_NAME        PARENT_ID
# Production           organizations/123456789012
# Non-Production       organizations/123456789012
# Shared-Services      organizations/123456789012
```

Org policies (Level 3, Module 04) and default IAM bindings attached at the
folder level inherit automatically to every project created underneath —
so "Production" can enforce `compute.vmExternalIpAccess: denyAll` and every
project in it inherits that without per-project setup. Module 05 of this
level covers folder/project structuring in depth.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud recommender recommendations list` | Pull automated cost/security/performance suggestions. |
| `gcloud recommender recommendations mark-claimed` | Record that you're acting on a recommendation. |
| `gcloud resource-manager folders create` | Build landing-zone folder structure under an org. |
| Five pillars (Ops, Security, Reliability, Cost, Performance) | The lens for every architecture review. |

## Exercise

Take the multi-tier CI/CD project from Level 3 Module 10 and score it
against all five pillars — one sentence each on what it does well and one
gap. Then run (or describe, if quota-limited) `gcloud recommender
recommendations list` against a real or sample project for both the
MachineTypeRecommender and IamRecommender, and note what pillar each
returned suggestion maps to.
