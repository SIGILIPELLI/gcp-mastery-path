# 10 · Project — Multi-Tier CI/CD Pipeline

This capstone wires together five pieces from this level into a real
delivery pipeline: a Shared VPC hosting a GKE cluster (Modules 01, 02),
Cloud Build deploying on every push (Module 06), a Cloud Workflow gating
the rollout (Module 03), and Cloud Trace/SCC watching the result (Modules
08, 09). The point is the integration, not new syntax.

## Architecture

```
        GitHub push to main
               │
        Cloud Build trigger
               │
     ┌─────────┴──────────┐
     │  build + push image │  → Artifact Registry (us-central1)
     └─────────┬──────────┘
               │
      Cloud Workflow: deploy-gate
      ┌────────┴─────────┐
      │ 1. deploy canary  │  → GKE (10% traffic)
      │ 2. wait + check   │  → Cloud Trace latency, error rate
      │ 3. switch:        │
      │    healthy → full │  → GKE (100% traffic)
      │    unhealthy → rollback
      └───────────────────┘
               │
      GKE cluster (Shared VPC host: net-host-project)
      Workload Identity → BigQuery export SA (read-only)
               │
      Security Command Center watches for drift/misconfig
```

A push triggers a build; the built image is deployed as a canary; a
workflow checks Trace-derived latency after a soak period; if healthy it
promotes to full traffic, otherwise it rolls back automatically — no human
in the loop for the common case, but every step is inspectable via `gcloud
builds log` / `gcloud workflows executions describe`.

## Step 1 — Shared VPC + GKE cluster

```bash
gcloud compute shared-vpc enable net-host-project
gcloud compute shared-vpc associated-projects add app-project \
  --host-project=net-host-project

gcloud container clusters create prod-cluster \
  --project=app-project \
  --network=projects/net-host-project/global/networks/shared-vpc \
  --subnetwork=projects/net-host-project/regions/us-central1/subnetworks/app-subnet \
  --region=us-central1 \
  --workload-pool=app-project.svc.id.goog
```

## Step 2 — Artifact Registry + Cloud Build trigger

```bash
gcloud artifacts repositories create app \
  --repository-format=docker --location=us-central1 --project=app-project

gcloud builds triggers create github \
  --repo-name=orders-service --repo-owner=my-org \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml \
  --project=app-project
```

```yaml
# cloudbuild.yaml
steps:
  - name: gcr.io/cloud-builders/docker
    args: [build, -t, "us-central1-docker.pkg.dev/$PROJECT_ID/app/orders:$SHORT_SHA", .]
  - name: gcr.io/cloud-builders/docker
    args: [push, "us-central1-docker.pkg.dev/$PROJECT_ID/app/orders:$SHORT_SHA"]
  - name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: gcloud
    args:
      - workflows
      - execute
      - deploy-gate
      - --location=us-central1
      - --data={"image":"us-central1-docker.pkg.dev/$PROJECT_ID/app/orders:$SHORT_SHA"}
```

The last step hands off to the Workflow rather than deploying directly —
Cloud Build's job ends at "image built and rollout requested," and the
canary/promote/rollback logic lives in one reviewable Workflow definition.

## Step 3 — Canary + health-gated promotion workflow

```yaml
# deploy-gate.yaml
main:
  params: [input]
  steps:
    - deploy_canary:
        call: googleapis.container.v1.projects.zones.clusters.get
        args: {}
        next: apply_canary
    - apply_canary:
        call: http.post
        args:
          url: ${"https://container.googleapis.com/..."}
          body:
            image: ${input.image}
            trafficPercent: 10
        result: canary_result
    - soak:
        call: sys.sleep
        args:
          seconds: 300
    - check_health:
        call: http.get
        args:
          url: https://monitoring.googleapis.com/v3/.../timeSeries
        result: health
    - decide:
        switch:
          - condition: ${health.body.errorRate < 0.01}
            next: promote_full
        next: rollback
    - promote_full:
        call: http.post
        args:
          url: ${"https://container.googleapis.com/..."}
          body:
            image: ${input.image}
            trafficPercent: 100
        next: end
    - rollback:
        call: http.post
        args:
          url: ${"https://container.googleapis.com/..."}
          body:
            trafficPercent: 0
        next: fail_deploy
    - fail_deploy:
        raise: "canary unhealthy, rolled back"
```

```bash
gcloud workflows deploy deploy-gate \
  --source=deploy-gate.yaml --location=us-central1 \
  --service-account=deploy-gate-sa@app-project.iam.gserviceaccount.com
```

The 5-minute `sys.sleep` soak is deliberately conservative — enough time
for Trace/latency metrics on the canary to become statistically meaningful
before the health check step queries them, rather than judging health off
the first few requests.

## Step 4 — Workload Identity for read-only BigQuery export

```bash
gcloud iam service-accounts create bq-export-gsa --project=app-project

gcloud projects add-iam-policy-binding app-project \
  --member="serviceAccount:bq-export-gsa@app-project.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataViewer"

gcloud iam service-accounts add-iam-policy-binding \
  bq-export-gsa@app-project.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:app-project.svc.id.goog[default/bq-export-ksa]"
```

## Step 5 — SCC watching for drift

```bash
gcloud scc findings list organizations/123456789012 \
  --filter='resourceName:"app-project" AND state="ACTIVE"'
```

Run this as a scheduled Cloud Build trigger on a nightly cron (or a Cloud
Scheduler job hitting a small Cloud Function) so config drift — like
someone manually opening a firewall rule during an incident and forgetting
to close it — surfaces automatically rather than at the next quarterly
audit.

## Cleanup

Tear down in reverse-dependency order: workflow, Cloud Build trigger, GKE
cluster, then detach the service project from Shared VPC before deleting
the host project's network — deleting the network first while a service
project is still attached fails with a dependency error.

```bash
gcloud workflows delete deploy-gate --location=us-central1 --project=app-project -q
gcloud builds triggers delete TRIGGER_ID --project=app-project -q
gcloud container clusters delete prod-cluster --region=us-central1 --project=app-project -q
gcloud compute shared-vpc associated-projects remove app-project --host-project=net-host-project
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud compute shared-vpc enable` / `associated-projects add` | Wire a service project into a Shared VPC host. |
| `gcloud builds triggers create github` | Auto-build on push to main. |
| `gcloud workflows deploy` / `execute` | Canary-gate the actual rollout logic. |
| `sys.sleep` in a workflow | Soak period before judging canary health. |
| Workload Identity KSA/GSA binding | Pod-level read-only BigQuery access, no keys. |
| `gcloud scc findings list --filter=` | Continuous drift/misconfig detection. |

## Stretch goals

- Replace the fixed 5-minute soak with a loop that polls Trace/Monitoring
  every 60 seconds and exits early once enough samples are healthy,
  capped at a max wait.
- Add a Slack/webhook notification step in the workflow's `rollback` branch
  so a failed canary pages someone instead of failing silently in a
  workflow execution log.
- Extend the SCC check into a hard gate: fail the Cloud Build pipeline if
  there are any `HIGH` severity active findings on the target project at
  deploy time.
- Add a second GKE cluster in another region and extend the workflow to
  canary-deploy to both before promoting either to full traffic.
