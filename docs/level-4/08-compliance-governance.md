# 08 · Compliance & Governance (Assured Workloads, Policy Intelligence)

Regulated workloads (government, healthcare, financial services) need
enforceable guarantees about data residency, personnel access, and
encryption — beyond what org policies alone express. **Assured Workloads**
packages those guarantees into a compliance-regime-specific folder;
**Policy Intelligence** helps you understand and tighten IAM at scale
before an auditor finds the gaps.

## Assured Workloads

```bash
gcloud assured workloads create \
  --organization=123456789012 \
  --location=us-central1 \
  --display-name="FedRAMP-Moderate-Workload" \
  --compliance-regime=FEDRAMP_MODERATE \
  --billing-account=012345-6789AB-CDEF01
```

```bash
gcloud assured workloads list --organization=123456789012 --location=us-central1
# DISPLAY_NAME                     COMPLIANCE_REGIME    STATE
# FedRAMP-Moderate-Workload        FEDRAMP_MODERATE     ACTIVE
```

Creating an Assured Workloads folder auto-applies a bundle of org policies
matching the chosen regime (e.g., data residency restricted to US regions,
restricted support personnel, mandatory CMEK for supported services) —
you don't hand-assemble the constraint list yourself; the regime template
does it, and projects created inside inherit it automatically.

```bash
gcloud assured workloads describe WORKLOAD_ID \
  --organization=123456789012 --location=us-central1 \
  --format="value(resourceSettings)"
```

**Gotcha — moving an existing project into an Assured Workloads folder is
restricted or disallowed for some regimes.** Assured Workloads is
generally meant to be the starting point for a workload's lifecycle, not a
retrofit — check `gcloud assured workloads` documentation for the specific
regime before assuming an existing prod project can simply be
"moved in" later; some regimes require projects to be created fresh inside
the workload folder from day one.

## Policy Intelligence: IAM Recommender and Policy Analyzer

**IAM Recommender** flags overprivileged grants based on actual usage:

```bash
gcloud recommender recommendations list \
  --project=my-project \
  --recommender=google.iam.policy.Recommender \
  --location=global \
  --format="table(content.overview, primaryImpact.category)"
```

```bash
# CONTENT.OVERVIEW                                        PRIMARY_IMPACT.CATEGORY
# Remove roles/editor from svc-legacy@my-project...        SECURITY
```

**Policy Analyzer** answers "who can do X" across the whole org — critical
for an audit question like "which identities can delete a BigQuery
dataset":

```bash
gcloud policy-intelligence query-activity \
  --activity-type=serviceAccountKeyLastAuthentication \
  --project=my-project
```

```bash
gcloud asset analyze-iam-policy \
  --organization=123456789012 \
  --full-resource-name="//bigquery.googleapis.com/projects/my-project/datasets/orders" \
  --permissions="bigquery.datasets.delete"
```

```bash
# principal: user:contractor@example.com
# role: roles/bigquery.admin
# ... (grants bigquery.datasets.delete transitively via this role)
```

Finding a contractor account with delete rights on a production dataset
via this query, rather than during an incident review, is the entire
value proposition of running Policy Analyzer proactively rather than
reactively.

**Gotcha — Policy Analyzer results reflect a snapshot, and propagation of
IAM changes takes time.** A binding removed minutes ago may still show in
results due to caching (`gcloud asset` queries against Cloud Asset
Inventory, which has its own refresh latency) — don't treat a query result
as instantaneous ground truth immediately after making a change; re-check
after a few minutes.

## Data residency and CMEK enforcement

```bash
gcloud org-policies set-policy resource-locations-policy.yaml --organization=123456789012
```

```yaml
# resource-locations-policy.yaml
name: organizations/123456789012/policies/gcp.resourceLocations
spec:
  rules:
    - values:
        allowedValues:
          - "in:us-locations"
```

```bash
gcloud kms keyrings create prod-keyring --location=us-central1
gcloud kms keys create prod-cmek-key --keyring=prod-keyring --location=us-central1 --purpose=encryption

gcloud storage buckets create gs://orders-regulated-data \
  --location=us-central1 \
  --default-encryption-key=projects/my-project/locations/us-central1/keyRings/prod-keyring/cryptoKeys/prod-cmek-key
```

**Gotcha — CMEK doesn't retroactively re-encrypt existing objects.**
Setting `--default-encryption-key` on a bucket only applies to objects
written *after* the change; pre-existing objects remain under
Google-managed encryption until explicitly rewritten
(`gsutil rewrite -k -r gs://bucket`). An auditor checking "is everything
CMEK-encrypted" will find gaps if this rewrite step is skipped after
retrofitting CMEK onto an existing bucket.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud assured workloads create --compliance-regime=` | Provision a regime-templated compliance folder. |
| `gcloud recommender ... google.iam.policy.Recommender` | Surface overprivileged IAM grants. |
| `gcloud asset analyze-iam-policy --full-resource-name= --permissions=` | Answer "who can do X" org-wide. |
| `gcloud org-policies set-policy gcp.resourceLocations` | Enforce data residency at org/folder scope. |
| `gsutil rewrite -k -r` | Re-encrypt existing objects after retrofitting CMEK. |

## Exercise

Write the command to create an Assured Workloads folder for a hypothetical
regime, and describe (from the regime template concept) what categories of
constraint it would auto-apply. Then write a `gcloud asset
analyze-iam-policy` command answering "who can delete objects in bucket
`orders-regulated-data`" and describe the caching/propagation caveat you'd
mention if someone questioned a stale-looking result.
