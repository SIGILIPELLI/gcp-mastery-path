# 02 · Kubernetes on GCP (GKE Advanced)

Level 2 got a GKE Autopilot cluster running a service. This module goes
deeper: node pool strategy, workload identity for pod-level GCP auth,
horizontal/vertical autoscaling tuned for real traffic, and multi-cluster
patterns for availability across regions.

## Standard vs. Autopilot, and node pools

Autopilot manages nodes for you; **Standard** mode gives you direct control
over node pools — useful when you need specific machine types, GPUs, or
spot VMs for cost.

```bash
gcloud container clusters create prod-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=e2-standard-4 \
  --enable-ip-alias

# Add a second pool for batch workloads on Spot VMs
gcloud container node-pools create batch-pool \
  --cluster=prod-cluster \
  --zone=us-central1-a \
  --machine-type=e2-standard-8 \
  --spot \
  --num-nodes=2 \
  --node-taints=workload=batch:NoSchedule
```

The taint forces only pods with a matching toleration onto the Spot pool,
keeping latency-sensitive workloads off nodes that can be reclaimed with
30 seconds' notice.

```bash
gcloud container node-pools list --cluster=prod-cluster --zone=us-central1-a
# NAME         MACHINE_TYPE   NODE_VERSION   NUM_NODES
# default-pool e2-standard-4  1.29.1-gke.x   3
# batch-pool   e2-standard-8  1.29.1-gke.x   2
```

**Gotcha — Spot node reclaim.** Spot VMs can be preempted at any time with
no SLA. Pods on a Spot pool need a `PodDisruptionBudget` and should be
stateless or checkpoint their own progress; anything writing to local disk
without replication will lose data on reclaim.

## Workload Identity

Workload Identity binds a Kubernetes ServiceAccount (KSA) to a Google
service account (GSA), so pods authenticate to GCP APIs without mounting a
key file.

```bash
gcloud container clusters update prod-cluster \
  --zone=us-central1-a \
  --workload-pool=my-project.svc.id.goog

gcloud iam service-accounts create orders-gsa

gcloud iam service-accounts add-iam-policy-binding orders-gsa@my-project.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:my-project.svc.id.goog[default/orders-ksa]"
```

```yaml
# orders-ksa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-ksa
  namespace: default
  annotations:
    iam.gke.io/gcp-service-account: orders-gsa@my-project.iam.gserviceaccount.com
```

Any pod running with `serviceAccountName: orders-ksa` now gets short-lived
credentials for `orders-gsa`'s IAM grants automatically — no `Secret`
holding a JSON key ever touches the cluster.

**Gotcha — namespace/KSA name is baked into the binding.** The binding above
only works for `default/orders-ksa` exactly. Deploying the same manifest
into a different namespace silently gets no credentials (calls fail with
403) unless you add a matching binding for that namespace too.

## Autoscaling: HPA, VPA, and cluster autoscaler together

```bash
kubectl autoscale deployment orders-api --cpu-percent=60 --min=3 --max=20
```

```bash
gcloud container clusters update prod-cluster \
  --zone=us-central1-a \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --node-pool=default-pool
```

HPA scales pod *count* based on CPU/memory/custom metrics; the cluster
autoscaler scales *node count* to fit the resulting pods. They operate on
different timescales — HPA reacts in seconds, cluster autoscaler takes
minutes to provision a new node — so a traffic spike can leave pods stuck
`Pending` briefly while nodes catch up.

**Gotcha — don't mix HPA and VPA on the same metric.** Running a
`VerticalPodAutoscaler` in `Auto` mode alongside an HPA that also watches
CPU creates a feedback loop: VPA resizes the pod, which changes the CPU
percentage HPA reacts to, which changes replica count, which changes
per-pod load VPA reacts to. Use VPA for memory/CPU *requests* sizing and HPA
for replica count on a different signal (e.g., requests-per-second) instead.

## Multi-cluster / regional resilience

For availability beyond one zone or region, run clusters in two regions
behind a **Multi Cluster Ingress (MCI)** or a global external load balancer
pointed at both:

```bash
gcloud container clusters create prod-us --region=us-central1 --num-nodes=2
gcloud container clusters create prod-eu --region=europe-west1 --num-nodes=2

gcloud container fleet ingress enable --project=my-project
gcloud container fleet memberships register prod-us-membership \
  --gke-cluster=us-central1/prod-us --enable-workload-identity
gcloud container fleet memberships register prod-eu-membership \
  --gke-cluster=europe-west1/prod-eu --enable-workload-identity
```

**Gotcha — fleet membership is required before MCI features work.** Both
clusters must be registered to the same GKE fleet; a cluster created and
left unregistered simply doesn't show up as an eligible backend, with no
error at the Ingress level — check `gcloud container fleet memberships
list` if a cluster seems to be ignored.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud container node-pools create --spot` | Add a cost-optimized, preemptible node pool. |
| `--workload-pool=PROJECT.svc.id.goog` | Enable Workload Identity on a cluster. |
| `iam.gke.io/gcp-service-account` annotation | Bind a KSA to a GSA for pod-level auth. |
| `kubectl autoscale deployment` | Configure HPA for a deployment. |
| `gcloud container clusters update --enable-autoscaling` | Configure cluster (node) autoscaling. |
| `gcloud container fleet memberships register` | Join a cluster to a fleet for multi-cluster features. |

## Exercise

Create a Standard-mode cluster with a default pool and a tainted Spot pool.
Enable Workload Identity, create a KSA/GSA binding for a `orders-ksa` in the
`default` namespace, and grant that GSA `roles/storage.objectViewer`.
Deploy a pod using `orders-ksa` and confirm (via `kubectl exec` + `gcloud
auth list` or a metadata-server curl) that it presents the GSA's identity,
not a key file. Then attach an HPA to a sample deployment and watch
`kubectl get hpa -w` while load-testing it.
