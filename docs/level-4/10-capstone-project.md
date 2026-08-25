# 10 · Capstone Project — Production-Grade Cloud Architecture

This capstone combines nearly everything from Levels 1-4 into one
architecture: a multi-project org structure (Module 05) hosting a
multi-region GKE-at-scale deployment (Module 04) behind a global load
balancer, streaming analytics through Eventarc/Dataflow/BigQuery (Modules
02-03), governed by Assured Workloads and Policy Intelligence (Module 08),
cost-controlled with CUDs and labels (Module 06), connected via Network
Connectivity Center (Module 07), and validated with chaos game days
(Module 09) — reviewed against the Architecture Framework's five pillars
(Module 01).

## Architecture

```
Organization
├── Folder: Production
│   ├── orders-prod (GKE us-central1 + europe-west1, via fleet)
│   ├── analytics-prod (BigQuery, Dataflow)
│   └── net-host-prod (Shared VPC host, NCC hub)
└── Folder: Shared-Services
    └── logging-billing-project (org-wide log sink, billing export)

                    example.com (Cloud DNS)
                             │
              Global external HTTP(S) Load Balancer
                    (Cloud CDN + Cloud Armor)
                    /                        \
        GKE us-central1                  GKE europe-west1
        "orders-api" (fleet member)      "orders-api" (fleet member)
        Workload Identity → BigQuery      Workload Identity → BigQuery
             │                                    │
             └───────────── Pub/Sub "orders-events" ─────────────┘
                                    │
                          Eventarc / Dataflow
                          (streaming aggregation)
                                    │
                    BigQuery (partitioned + clustered)
                                    │
                     Looker Studio / analytics consumers

  Cross-cutting: Assured Workloads folder policies, NCC hub linking both
  regions' VPCs + Shared-Services, org-wide audit log sink, CUD covering
  baseline GKE node footprint, mandatory cost-center labels enforced in CI.
```

## Step 1 — Org structure and Shared VPC

```bash
gcloud resource-manager folders create --display-name="Production" --organization=123456789012
gcloud resource-manager folders create --display-name="Shared-Services" --organization=123456789012

gcloud projects create net-host-prod --folder=PROD_FOLDER_ID
gcloud projects create orders-prod --folder=PROD_FOLDER_ID
gcloud projects create analytics-prod --folder=PROD_FOLDER_ID
gcloud projects create logging-billing-project --folder=SHARED_FOLDER_ID

gcloud compute shared-vpc enable net-host-prod
gcloud compute shared-vpc associated-projects add orders-prod --host-project=net-host-prod
```

## Step 2 — NCC hub linking regional VPCs

```bash
gcloud network-connectivity hubs create global-hub --project=net-host-prod

gcloud network-connectivity spokes linked-vpc-network create us-spoke \
  --hub=global-hub --vpc-network=shared-vpc --region=us-central1 --project=net-host-prod
gcloud network-connectivity spokes linked-vpc-network create eu-spoke \
  --hub=global-hub --vpc-network=shared-vpc --region=europe-west1 --project=net-host-prod
```

## Step 3 — Multi-region GKE fleet

```bash
gcloud container clusters create orders-us --project=orders-prod \
  --region=us-central1 --network=projects/net-host-prod/global/networks/shared-vpc \
  --subnetwork=projects/net-host-prod/regions/us-central1/subnetworks/app-subnet \
  --workload-pool=orders-prod.svc.id.goog --enable-dataplane-v2

gcloud container clusters create orders-eu --project=orders-prod \
  --region=europe-west1 --network=projects/net-host-prod/global/networks/shared-vpc \
  --subnetwork=projects/net-host-prod/regions/europe-west1/subnetworks/app-subnet \
  --workload-pool=orders-prod.svc.id.goog --enable-dataplane-v2

gcloud container fleet memberships register orders-us-membership \
  --gke-cluster=us-central1/orders-us --project=orders-prod
gcloud container fleet memberships register orders-eu-membership \
  --gke-cluster=europe-west1/orders-eu --project=orders-prod
```

Global external Application Load Balancer fronting both clusters via
Multi Cluster Ingress, exactly as in Level 3 Module 07's failover pattern —
a zone or region loss fails traffic over via health checks, no DNS TTL
wait.

## Step 4 — Event pipeline into analytics

```bash
gcloud pubsub topics create orders-events --project=orders-prod

gcloud eventarc triggers create orders-to-pubsub-bridge \
  --location=us-central1 --project=orders-prod \
  --destination-run-service=orders-events-bridge \
  --event-filters="type=google.cloud.pubsub.topic.v1.messagePublished"
```

```python
# Dataflow streaming job (analytics-prod)
with beam.Pipeline(options=PipelineOptions(
        runner="DataflowRunner", project="analytics-prod",
        region="us-central1", streaming=True)) as p:
    (
        p
        | beam.io.ReadFromPubSub(topic="projects/orders-prod/topics/orders-events")
        | beam.Map(parse_event)
        | beam.WindowInto(beam.window.FixedWindows(60))
        | beam.io.WriteToBigQuery("analytics-prod:orders.event_counts",
              write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND)
    )
```

```sql
CREATE TABLE orders.event_counts (
  window_start TIMESTAMP, event_type STRING, count INT64
)
PARTITION BY DATE(window_start)
CLUSTER BY event_type;
```

Workload Identity binds `orders-api`'s KSA in both clusters to a GSA with
`roles/pubsub.publisher` scoped to `orders-events` only — no broader
project access, following the least-privilege pattern from Level 3 Module
04.

## Step 5 — Governance and cost guardrails

```bash
gcloud assured workloads create --organization=123456789012 --location=us-central1 \
  --display-name="orders-prod-compliance" --compliance-regime=FEDRAMP_MODERATE \
  --billing-account=012345-6789AB-CDEF01

gcloud logging sinks create org-audit-sink \
  bigquery.googleapis.com/projects/logging-billing-project/datasets/org_logs \
  --organization=123456789012 --include-children \
  --log-filter='logName:"cloudaudit.googleapis.com"'

gcloud compute commitments create orders-baseline-cud \
  --project=orders-prod --region=us-central1 \
  --plan=twelve-month --resources=vcpu=64,memory=256GB
```

CI enforces `cost-center` and `team` labels on every Terraform-managed
resource (Level 4 Module 06) before merge; `gcloud recommender` runs
weekly against both clusters' projects to catch IAM and machine-type drift.

## Step 6 — Resilience validation

```bash
# Quarterly game day: simulate us-central1 GKE zone loss
gcloud container clusters resize orders-us --region=us-central1 \
  --node-pool=default-pool --num-nodes=0 --project=orders-prod

gcloud compute backend-services get-health orders-global-backend --global
# Confirm traffic shifts entirely to orders-eu, latency/error SLO holds

gcloud container clusters resize orders-us --region=us-central1 \
  --node-pool=default-pool --num-nodes=3 --project=orders-prod
```

## Architecture Framework review

| Pillar | How this design addresses it |
|---|---|
| Operational Excellence | Fleet-registered clusters, Config Sync (Module 04) for consistent config, org-wide audit sink |
| Security/Privacy/Compliance | Assured Workloads folder, Workload Identity (no keys), least-privilege pub/sub role |
| Reliability | Multi-region GKE + global LB failover, validated quarterly via game day |
| Cost Optimization | CUD sized to baseline, mandatory labels, weekly recommender review |
| Performance Optimization | Partitioned/clustered BigQuery tables, windowed Dataflow aggregation, CDN on the LB |

## Cleanup

Tear down in reverse order: game-day validation stops, Dataflow job
cancelled, Eventarc trigger and Pub/Sub topic deleted, fleet memberships
unregistered, GKE clusters deleted, NCC spokes/hub deleted, Shared VPC
association removed, projects deleted last.

```bash
gcloud dataflow jobs cancel JOB_ID --region=us-central1 --project=analytics-prod
gcloud container fleet memberships unregister orders-us-membership --project=orders-prod
gcloud container clusters delete orders-us --region=us-central1 --project=orders-prod -q
gcloud container clusters delete orders-eu --region=europe-west1 --project=orders-prod -q
gcloud network-connectivity spokes linked-vpc-network delete us-spoke --project=net-host-prod -q
gcloud network-connectivity hubs delete global-hub --project=net-host-prod -q
```

## Stretch goals

- Add a second Assured Workloads-governed analytics-only region for data
  residency requirements distinct from the compute regions.
- Replace the fixed-CUD sizing with an automated quarterly review that
  compares committed capacity against `MachineTypeRecommender` output and
  proposes a resize.
- Extend the chaos game day to inject a Pub/Sub subscription backlog
  (pause the Dataflow job briefly) and verify the dead-letter/backpressure
  handling holds without data loss.
- Add a second compliance regime folder alongside the first and compare
  which org policies differ between them, documenting the delta.
