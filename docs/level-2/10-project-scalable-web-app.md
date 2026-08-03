# 10 · Project — Scalable Web App

This capstone combines four pieces from this level into one working system:
a GKE-hosted API (Module 01), an event pushed to Pub/Sub for async work
(Module 04), order data in Firestore (Module 05), and a CDN-fronted, custom
domain served over Cloud DNS with Cloud Armor protection (Modules 03, 08).
Nothing here is new syntax — it's the pattern of wiring modules together
into an architecture, which is the actual skill this level has been
building toward.

## Architecture

```
                     example.com (Cloud DNS)
                              │
                  Global HTTP(S) Load Balancer
                    (Cloud CDN + Cloud Armor)
                              │
                    GKE Autopilot cluster
                     "orders-api" Service
                              │
              ┌───────────────┴───────────────┐
              │                                │
        Firestore (Native)                Pub/Sub "orders" topic
        orders collection                        │
                                          "orders-fulfillment-sub"
                                          → fulfillment worker (GKE)
```

A request hits the load balancer → CDN serves static assets straight from
cache; API calls pass through to the GKE-hosted `orders-api`. The API writes
the order to Firestore and publishes an event to Pub/Sub, then returns
immediately — a separate fulfillment worker consumes that event
asynchronously, so the customer-facing request never waits on fulfillment
logic.

## Step 1 — GKE cluster and orders-api

```bash
gcloud container clusters create-auto capstone-cluster \
  --region=us-central1

gcloud container clusters get-credentials capstone-cluster \
  --region=us-central1
```

```yaml
# orders-api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api
    spec:
      serviceAccountName: orders-api-ksa
      containers:
        - name: orders-api
          image: us-docker.pkg.dev/gcp-mastery-path-123/repo/orders-api:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: orders-api-svc
spec:
  type: NodePort
  selector:
    app: orders-api
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f orders-api-deployment.yaml
kubectl get pods
# NAME                          READY   STATUS    RESTARTS   AGE
# orders-api-7d9f8c5b4d-2k9p1   1/1     Running   0          45s
# orders-api-7d9f8c5b4d-x8m4q   1/1     Running   0          45s
```

`type: NodePort` (not `LoadBalancer`) is deliberate — traffic reaches this
Service through a GKE Ingress in Step 4, not a per-Service external IP,
avoiding the "one billed forwarding rule per Service" gotcha from Module 01.

## Step 2 — Firestore for order data

```bash
gcloud firestore databases create --location=nam5 --type=firestore-native
```

```bash
gcloud iam service-accounts create orders-api-gsa \
  --display-name="Orders API GCP identity"

gcloud projects add-iam-policy-binding gcp-mastery-path-123 \
  --member="serviceAccount:orders-api-gsa@gcp-mastery-path-123.iam.gserviceaccount.com" \
  --role="roles/datastore.user"

kubectl create serviceaccount orders-api-ksa

gcloud iam service-accounts add-iam-policy-binding \
  orders-api-gsa@gcp-mastery-path-123.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:gcp-mastery-path-123.svc.id.goog[default/orders-api-ksa]"

kubectl annotate serviceaccount orders-api-ksa \
  iam.gke.io/gcp-service-account=orders-api-gsa@gcp-mastery-path-123.iam.gserviceaccount.com
```

This is Workload Identity from Module 01 — `orders-api` pods authenticate to
Firestore as `orders-api-gsa` with no key material on disk, using
`roles/datastore.user` (Firestore's IAM role name is inherited from its
Datastore-mode history).

## Step 3 — Pub/Sub for async fulfillment

```bash
gcloud pubsub topics create orders

gcloud pubsub subscriptions create orders-fulfillment-sub \
  --topic=orders \
  --ack-deadline=60 \
  --dead-letter-topic=orders-dlq \
  --max-delivery-attempts=5

gcloud pubsub topics create orders-dlq

gcloud projects add-iam-policy-binding gcp-mastery-path-123 \
  --member="serviceAccount:orders-api-gsa@gcp-mastery-path-123.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"
```

`orders-api`'s request handler does two things per order, in this order:
write to Firestore (the durable record), then publish to `orders` (the async
trigger) — writing the durable record first means a Pub/Sub publish failure
never loses the order itself, only delays fulfillment, which can be
re-triggered from the Firestore record if needed.

The fulfillment worker is a second GKE Deployment pulling from
`orders-fulfillment-sub`, following the dead-letter pattern from Module 04
so a bad order doesn't loop forever.

## Step 4 — Ingress, load balancer, CDN, DNS

```yaml
# orders-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orders-ingress
  annotations:
    kubernetes.io/ingress.global-static-ip-name: web-lb-ip
    networking.gke.io/managed-certificates: orders-cert
spec:
  defaultBackend:
    service:
      name: orders-api-svc
      port:
        number: 80
---
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: orders-cert
spec:
  domains:
    - example.com
```

```bash
gcloud compute addresses create web-lb-ip --global

kubectl apply -f orders-ingress.yaml

kubectl get ingress orders-ingress
# NAME             CLASS  HOSTS  ADDRESS         PORTS  AGE
# orders-ingress   <none> *      34.117.20.10    80     3m
```

A GKE Ingress provisions the same global HTTP(S) load balancer chain from
Module 02 (backend service, URL map, proxy, forwarding rule) automatically —
enabling CDN and Cloud Armor is then the identical `gcloud compute
backend-services update` call from Modules 08, just targeting the
Ingress-managed backend service name instead of one you created by hand:

```bash
gcloud compute backend-services list --format="value(name)"
# k8s1-a1b2c3d4-default-orders-api-svc-80-e5f6g7h8

gcloud compute backend-services update k8s1-a1b2c3d4-default-orders-api-svc-80-e5f6g7h8 \
  --global \
  --enable-cdn \
  --security-policy=web-security-policy
```

```bash
gcloud dns record-sets transaction start --zone=example-com
gcloud dns record-sets transaction add 34.117.20.10 \
  --name="example.com." --ttl=300 --type=A --zone=example-com
gcloud dns record-sets transaction execute --zone=example-com
```

## Step 5 — Verify end to end

```bash
curl -X POST https://example.com/orders \
  -H "Content-Type: application/json" \
  -d '{"item": "widget", "qty": 3}'
# {"orderId": "A1002", "status": "received"}

gcloud pubsub subscriptions pull orders-fulfillment-sub --auto-ack --limit=1
# {"orderId": "A1002", "item": "widget", "qty": 3}
```

A successful run shows: the order returned instantly (Firestore write, then
publish — not waiting on fulfillment), the fulfillment subscription received
the event, and a second `curl` for a static asset shows `X-Cache: HIT` on
repeat.

## Step 6 — Cost guardrail

```bash
gcloud billing budgets create \
  --billing-account=012345-6789AB-CDEF01 \
  --display-name="Capstone Budget" \
  --budget-amount=25USD \
  --threshold-rule=percent=0.5 \
  --threshold-rule=percent=1.0
```

Wiring a budget in as the very last step (Module 09) closes the loop on
everything provisioned above — a GKE cluster, a load balancer, and Cloud
Armor together are the most expensive combination in this course, and it's
easy to forget one running after the exercise is "done."

## Stretch goals

- **Multi-region.** Add a second GKE cluster in another region behind the
  same global load balancer, and confirm the global LB routes each client to
  its nearest healthy backend.
- **Blue/green fulfillment worker.** Deploy a v2 fulfillment worker
  alongside v1, and use `kubectl` rollout controls (Module 01) to shift
  traffic gradually while watching the dead-letter topic for any errors.
- **Cost dashboard.** Wire billing export to BigQuery (Module 09) and build
  a query that breaks this capstone's spend down by label
  (`app=orders-api`, `app=fulfillment-worker`) to see which piece actually
  costs the most.
- **DNSSEC + Cloud Armor WAF.** Turn on DNSSEC for the zone and add a
  preconfigured SQLi/XSS Cloud Armor rule set, then use a benign test
  payload to confirm it's actually blocked before assuming it works.

## Cleanup

```bash
kubectl delete -f orders-ingress.yaml
kubectl delete -f orders-api-deployment.yaml

gcloud compute addresses delete web-lb-ip --global --quiet
gcloud pubsub subscriptions delete orders-fulfillment-sub
gcloud pubsub topics delete orders orders-dlq

gcloud dns record-sets transaction start --zone=example-com
gcloud dns record-sets transaction remove 34.117.20.10 \
  --name="example.com." --ttl=300 --type=A --zone=example-com
gcloud dns record-sets transaction execute --zone=example-com

gcloud container clusters delete capstone-cluster --region=us-central1 --quiet
gcloud billing budgets delete <budget-id> --billing-account=012345-6789AB-CDEF01
```

Delete in this order — Ingress/Service first (so GKE releases the backend
service and forwarding rule cleanly), then the reserved IP, then the
cluster — to avoid orphaned load-balancer resources billing after the
cluster is gone.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud container clusters create-auto` | Provision the GKE Autopilot cluster hosting both services. |
| Workload Identity binding + KSA annotation | Let pods authenticate to Firestore/Pub/Sub with no key files. |
| `gcloud pubsub subscriptions create --dead-letter-topic=` | Make the async fulfillment path failure-safe. |
| `kubectl apply -f orders-ingress.yaml` (with `ManagedCertificate`) | Provision the global LB + managed TLS cert from GKE. |
| `gcloud compute backend-services update --enable-cdn --security-policy=` | Attach CDN + Cloud Armor to the Ingress-managed backend. |
| `gcloud dns record-sets transaction ...` | Point the custom domain at the load balancer's IP. |
| `gcloud billing budgets create` | Guardrail the whole stack's spend. |

## Exercise

Build the full chain: GKE cluster with `orders-api`, Firestore for order
storage, a Pub/Sub topic with a dead-lettered fulfillment subscription, a
GKE Ingress with CDN and a Cloud Armor policy attached, and a custom domain
pointed at the resulting IP. Place an order via `curl`, confirm it's readable
in Firestore, confirm the fulfillment subscription receives the event, and
confirm a static asset shows `X-Cache: HIT` on a second request. Then tear
everything down in the order shown in Cleanup and confirm with `gcloud
compute forwarding-rules list` and `gcloud container clusters list` that
nothing billable remains.
