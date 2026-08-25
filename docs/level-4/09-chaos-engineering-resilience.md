# 09 · Chaos Engineering & Resilience Testing

Level 3's DR module tested failover by hand (pulling a backend). Chaos
engineering formalizes this: deliberately injecting failure into a
production-like system, on a schedule, to find weaknesses before an
uncontrolled outage does. This module covers how to run chaos experiments
safely on GCP.

## Principles before tooling

1. Define a **steady-state hypothesis** first — a measurable normal (e.g.,
   "p99 latency under 400ms, error rate under 0.5%").
2. Inject one failure at a time, in a pre-agreed **blast radius** (a
   specific service, region, or percentage of traffic — never "everything,
   everywhere").
3. Always have a **rollback trigger** and a person watching dashboards live
   — chaos experiments in production need an abort button, not just an
   end time.
4. Run in staging first, and only graduate an experiment to production once
   the failure mode and remediation are well understood.

## Zonal/regional failure injection

Simulate a zone outage by draining a GKE node pool or an MIG's zone:

```bash
gcloud compute instance-groups managed resize orders-mig-us-central1-a \
  --size=0 --zone=us-central1-a
```

```bash
gcloud compute backend-services get-health orders-backend --global
# backend: .../orders-mig-us-central1-a   healthState: UNHEALTHY
# backend: .../orders-mig-us-central1-b   healthState: HEALTHY
```

Watch the global load balancer redirect traffic to the remaining healthy
zone, and confirm your monitoring actually pages on the resulting
zone-level anomaly rather than staying silent because aggregate metrics
still look fine.

```bash
# Restore
gcloud compute instance-groups managed resize orders-mig-us-central1-a \
  --size=3 --zone=us-central1-a
```

**Gotcha — resizing to 0 is destructive to in-flight state on those
instances.** For anything stateful, drain connections gracefully first
(remove from the backend service, wait for connection draining timeout,
*then* resize) rather than yanking capacity immediately — otherwise you're
testing "what happens when we lose data," not "what happens when a zone
fails," which is a different and less useful experiment.

## Dependency failure injection

Simulate a downstream dependency (e.g., a database) becoming slow or
unavailable, without actually taking it down — using a service mesh fault
injection or an application-level feature flag:

```yaml
# Istio/Cloud Service Mesh VirtualService fault injection
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inventory-svc-fault
spec:
  hosts: [inventory-svc]
  http:
    - fault:
        delay:
          percentage:
            value: 50
          fixedDelay: 3s
      route:
        - destination:
            host: inventory-svc
```

```bash
kubectl apply -f inventory-fault.yaml
# ... observe orders-api behavior with 50% of inventory calls delayed 3s ...
kubectl delete -f inventory-fault.yaml
```

This answers "does `orders-api` time out and degrade gracefully, or does it
pile up threads and cascade-fail the whole service" — a question that's
much cheaper to answer deliberately than to discover during a real
`inventory-svc` incident.

**Gotcha — fault injection at the mesh layer only affects traffic that
goes through the mesh.** A service calling `inventory-svc` via its
internal ClusterIP directly, bypassing the mesh sidecar, won't see the
injected fault at all — verify the fault is actually being hit (check
`inventory-svc-fault`'s effect in request traces) before concluding a
service "handled it fine."

## Resource exhaustion experiments

```bash
kubectl run stress-test --image=polinux/stress --restart=Never \
  --limits=cpu=2,memory=2Gi \
  -- stress --cpu 4 --vm 2 --vm-bytes 1G --timeout 120s
```

```bash
kubectl top pods -n team-orders
# NAME              CPU(cores)   MEMORY(bytes)
# stress-test       2000m        2048Mi
```

Running this alongside real workloads in a namespace with a
`ResourceQuota` (Level 4, Module 04) validates that the quota and pod
resource limits actually prevent one runaway pod from starving its
neighbors — rather than assuming the quota configuration works because it
was applied without error.

**Gotcha — `stress` requesting more than its declared `--limits` gets
OOMKilled by the kubelet, which is the correct/expected outcome, not a
test failure.** The experiment is validating containment, not trying to
avoid the OOMKill — confusing "the pod got killed" with "the experiment
failed" is a common misread of results.

## Game days: scheduling and scope

```bash
gcloud scheduler jobs create http quarterly-gameday-reminder \
  --schedule="0 9 1 */3 *" \
  --uri=https://chat.googleapis.com/... \
  --message-body='{"text":"Quarterly game day scheduled — see runbook"}'
```

A recurring calendar cadence (quarterly is common) keeps chaos testing from
being a one-time exercise that's never repeated after the initial "we did
resilience testing" checkbox — infrastructure changes continuously, and a
mitigation that worked six months ago may no longer hold after a
dependency was refactored.

## Cheat sheet

| Concept / Command | Purpose |
|---|---|
| Steady-state hypothesis first | Define "normal" before injecting failure. |
| `gcloud compute instance-groups managed resize --size=0` | Simulate a zonal capacity loss. |
| Istio/mesh `VirtualService` fault injection | Simulate downstream latency/errors without real outages. |
| `kubectl run ... stress` + `ResourceQuota` | Validate resource isolation actually holds under load. |
| Scheduled game days | Keep chaos testing a recurring practice, not a one-off. |

## Exercise

Define a steady-state hypothesis for a sample service (specific latency
and error-rate numbers). Write the exact command sequence to simulate a
zonal MIG failure against that service (drain, resize to zero, observe
backend health, restore), and describe what dashboard/alert you'd expect
to fire during the experiment. Then design one dependency-fault-injection
experiment (mesh-based) and state the pass/fail criterion in terms of the
steady-state hypothesis you defined.
