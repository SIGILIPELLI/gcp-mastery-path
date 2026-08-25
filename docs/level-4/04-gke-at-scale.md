# 04 · Kubernetes at Scale (GKE Production Patterns)

Level 3's GKE Advanced module covered node pools, Workload Identity, and
autoscaling for one cluster. At production scale the questions shift:
multi-tenancy on shared clusters, release safety across hundreds of
deployments, and cost control across a fleet rather than one cluster.

## Multi-tenancy: namespaces, quotas, and network policy

Shared clusters across teams need hard boundaries or one team's bug starves
everyone else.

```bash
kubectl create namespace team-orders
kubectl create namespace team-payments
```

```yaml
# team-orders-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-orders-quota
  namespace: team-orders
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    pods: "50"
```

```yaml
# deny-cross-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: team-orders
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: {}   # only pods within team-orders namespace
```

```bash
kubectl apply -f team-orders-quota.yaml -f deny-cross-namespace.yaml
kubectl describe resourcequota team-orders-quota -n team-orders
# Used:  requests.cpu: 12, requests.memory: 24Gi, pods: 31
# Hard:  requests.cpu: 20, requests.memory: 40Gi, pods: 50
```

**Gotcha — GKE's network policy enforcement (Calico/Dataplane V2) must be
explicitly enabled at cluster creation**, or `NetworkPolicy` objects apply
with no error but also no effect:

```bash
gcloud container clusters create shared-cluster \
  --enable-dataplane-v2 \
  --zone=us-central1-a
```

An existing cluster created without this needs a (disruptive, requires
node pool recreation) migration — plan for it upfront on any
multi-tenant cluster.

## Progressive delivery: canary and blue/green with GKE

Beyond a bare `kubectl rollout`, production releases typically use a
service mesh or a controller like **Argo Rollouts** or GKE's built-in
traffic-split via Gateway API for percentage-based canaries.

```yaml
# rollout.yaml (Argo Rollouts CRD)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: orders-api
spec:
  replicas: 6
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 300}
        - setWeight: 50
        - pause: {duration: 300}
        - setWeight: 100
  selector:
    matchLabels: {app: orders-api}
  template:
    metadata:
      labels: {app: orders-api}
    spec:
      containers:
        - name: orders-api
          image: us-central1-docker.pkg.dev/my-project/app/orders:v12
```

```bash
kubectl argo rollouts get rollout orders-api --watch
# NAME            KIND        STATUS     AGE  INFO
# ⟳ orders-api    Rollout     Progressing 2m
# └──# revision:2
#    └──⧉ orders-api-6f9...   ReplicaSet  ● Healthy  10%
```

```bash
kubectl argo rollouts promote orders-api      # manually advance a paused step
kubectl argo rollouts abort orders-api        # rollback mid-rollout
```

**Gotcha — canary analysis needs a real metric provider wired in**, or the
`pause` steps are just timers with no actual health judgment — pair
`AnalysisTemplate` resources querying Cloud Monitoring/Trace (Level 3,
Module 08) so a canary that's silently erroring more doesn't get
auto-promoted just because the clock ran out.

## Cluster fleet management

Beyond one cluster, `gcloud container fleet` (Level 3 touched registration)
adds **Config Sync** and **Policy Controller** for keeping many clusters
consistent from one source of truth.

```bash
gcloud container fleet config-management apply \
  --membership=prod-us-membership \
  --config=config-sync.yaml
```

```yaml
# config-sync.yaml
applySpecVersion: 1
spec:
  configSync:
    sourceFormat: unstructured
    git:
      syncRepo: https://github.com/my-org/fleet-config
      syncBranch: main
      secretType: none
```

```bash
gcloud container fleet config-management status
# NAME                  STATUS
# prod-us-membership    SYNCED
# prod-eu-membership    SYNCED
```

Every registered cluster now pulls the same manifests from one git repo —
a `NetworkPolicy` or `ResourceQuota` change merged once propagates
everywhere, instead of `kubectl apply` run manually per cluster (and
inevitably drifting).

**Gotcha — Config Sync only pulls; it doesn't push local changes back or
warn loudly about manual drift beyond marking `NOT_SYNCED`.** A `kubectl
edit` against a Config-Sync-managed resource gets silently reverted on the
next sync interval — which is intended behavior, but surprises people
debugging "why did my change disappear."

## Cost control at fleet scale

```bash
gcloud container clusters update prod-cluster \
  --zone=us-central1-a \
  --enable-vertical-pod-autoscaling

gcloud recommender recommendations list \
  --project=my-project \
  --recommender=google.container.DiagnosisRecommender \
  --location=us-central1-a
```

Bin-packing efficiency matters more at fleet scale than per-cluster tuning
— GKE's cluster autoscaler with `--autoscaling-profile=optimize-utilization`
(vs. the default `balanced`) more aggressively consolidates workloads onto
fewer nodes, trading a little scheduling latency for meaningfully lower
node-hours across a large fleet.

## Cheat sheet

| Command | Purpose |
|---|---|
| `--enable-dataplane-v2` | Required at cluster creation for NetworkPolicy enforcement. |
| `ResourceQuota` per namespace | Hard-cap CPU/memory/pod count per team. |
| Argo Rollouts `canary.steps` | Weighted, paused, metric-gated progressive delivery. |
| `gcloud container fleet config-management apply` | GitOps-sync manifests across a cluster fleet. |
| `--autoscaling-profile=optimize-utilization` | Trade scheduling latency for higher bin-packing efficiency. |

## Exercise

Design a two-team shared cluster: two namespaces, a `ResourceQuota` per
namespace, and a `NetworkPolicy` denying cross-namespace ingress. Confirm
`--enable-dataplane-v2` is required and note what happens (silently) if
it's omitted. Then sketch an Argo Rollouts canary spec for a service with
three weighted steps and a 5-minute pause each, and describe what metric
you'd wire into an `AnalysisTemplate` to make the pauses actually
health-gated rather than pure timers.
