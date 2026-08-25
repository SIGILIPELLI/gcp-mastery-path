# 07 · Multi-Region & Disaster Recovery

A single-region deployment is fine until that region has a bad day. This
module covers designing for regional failure: choosing an RTO/RPO target,
replicating data across regions, and failing traffic over — plus the
GCP-specific quirks that make multi-region harder than "just deploy twice."

## RTO/RPO: pick targets before architecture

- **RTO** (Recovery Time Objective): how long can the service be down.
- **RPO** (Recovery Point Objective): how much data can you afford to lose.

| Pattern | Typical RTO | Typical RPO | Cost |
|---|---|---|---|
| Backup/restore | Hours | Hours (since last backup) | Low |
| Warm standby (scaled-down replica) | Minutes | Seconds–minutes | Medium |
| Active-active (both regions serving live traffic) | Seconds | Near-zero | High |

Committing to active-active for a service that only needs hours of RTO is
wasted spend; conversely a payments ledger that needs seconds-level RPO
can't be satisfied by nightly backups. Pick the pattern the business
requirement actually demands.

## Data replication

**Cloud SQL cross-region read replica** (warm standby pattern):

```bash
gcloud sql instances create orders-db-replica-eu \
  --master-instance-name=orders-db-primary \
  --region=europe-west1 \
  --tier=db-custom-4-16384
```

```bash
gcloud sql instances describe orders-db-replica-eu --format="value(state,region)"
# RUNNABLE  europe-west1
```

Promoting the replica during a real failover:

```bash
gcloud sql instances promote-replica orders-db-replica-eu
```

**Gotcha — promotion is one-way and irreversible.** Once promoted, the
replica becomes an independent primary; it does not automatically
re-establish replication back to the original region. Rebuilding
`us-central1` as a new replica of the newly-promoted `europe-west1` primary
is a separate, manual step — plan the runbook for that, not just the
initial failover.

**Spanner** sidesteps this entirely for workloads that can afford it —
multi-region Spanner configs replicate synchronously with automatic
failover and no promote step, at higher cost and with write-latency
implications from cross-region consensus:

```bash
gcloud spanner instances create orders-spanner \
  --config=nam3 \
  --nodes=3 \
  --description="Multi-region orders DB"
```

`nam3` is a predefined multi-region config spanning several US regions with
a designated leader region for lowest write latency; `gcloud spanner
instance-configs list` shows all available configs.

**GCS** multi-region/dual-region buckets replicate objects across
locations automatically at the storage layer:

```bash
gsutil mb -l NAM4 gs://orders-assets-dr
gsutil rewrite -r gs://orders-assets-dr  # backfill existing objects after enabling turbo replication
```

## Traffic failover

A global external Application Load Balancer with backends in two regions
fails traffic over automatically via health checks — no DNS TTL wait, since
it's one anycast IP:

```bash
gcloud compute backend-services create orders-backend \
  --global \
  --health-checks=orders-health-check

gcloud compute backend-services add-backend orders-backend \
  --global \
  --instance-group=orders-us-central1-mig \
  --instance-group-region=us-central1

gcloud compute backend-services add-backend orders-backend \
  --global \
  --instance-group=orders-europe-west1-mig \
  --instance-group-region=europe-west1
```

```bash
gcloud compute backend-services get-health orders-backend --global
# backend: .../orders-us-central1-mig
#   healthState: HEALTHY
# backend: .../orders-europe-west1-mig
#   healthState: HEALTHY
```

If `us-central1`'s backend fails health checks, the load balancer routes
100% of traffic to `europe-west1` within the health-check's unhealthy
threshold window (seconds, not the DNS-TTL-driven minutes of older
multi-region approaches) — no client-side change needed.

**Gotcha — health checks validate the LB path, not data consistency.** A
region can be "healthy" from the load balancer's perspective (HTTP 200 on
`/healthz`) while its read replica is lagging by minutes. Failing over
traffic doesn't fix a data-freshness problem — that's a separate check your
runbook needs (e.g., alert on replication lag before it becomes a silent
failover-to-stale-data situation).

## Testing DR without waiting for a real outage

```bash
# Simulate regional failure: pull one backend out deliberately
gcloud compute backend-services remove-backend orders-backend \
  --global \
  --instance-group=orders-us-central1-mig \
  --instance-group-region=us-central1
```

Run this as a scheduled game-day exercise, not just tabletop planning —
watch actual latency/error-rate dashboards during the drill and re-add the
backend afterward. A DR plan that's never been executed is a DR plan that
doesn't actually work yet.

**Gotcha — quota headroom in the failover region.** The standby region
needs enough quota (CPU, IP addresses, persistent disk) provisioned *ahead
of time* to absorb 100% of traffic, not just its normal warm-standby
fraction. `gcloud compute regions describe europe-west1
--format="value(quotas)"` shows current headroom — a failover that trips a
quota ceiling mid-incident turns a regional outage into a total outage.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud sql instances create --master-instance-name=` | Create a cross-region Cloud SQL read replica. |
| `gcloud sql instances promote-replica` | Failover: promote a replica to an independent primary (one-way). |
| `gcloud spanner instances create --config=nam3` | Multi-region Spanner with automatic synchronous failover. |
| `gcloud compute backend-services add-backend --instance-group-region=` | Add a regional backend to a global LB. |
| `gcloud compute backend-services get-health` | Check per-backend health across regions. |
| Game-day: `remove-backend` then re-add | Rehearse failover without a real outage. |

## Exercise

Provision (or plan on paper, given no real credentials) a Cloud SQL
primary in one region and a read replica in another. Write the exact
`gcloud` command sequence for a full failover runbook: promote the replica,
repoint the application's connection string, and re-establish replication
back to the original region once it recovers. State your chosen RTO/RPO
target and justify which pattern (backup/restore, warm standby,
active-active) it implies.
