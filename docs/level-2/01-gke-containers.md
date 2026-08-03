# 01 · Containers on GCP (GKE Basics)

Level 1 ran workloads as VMs (Compute Engine) and single containers (Cloud
Run). **Google Kubernetes Engine (GKE)** is the step in between: a managed
Kubernetes control plane that schedules many containers across a pool of
nodes, restarts them when they crash, and load-balances traffic between
replicas. This module creates a cluster, deploys a containerized app, and
exposes it to the internet.

## Autopilot vs. Standard

GKE has two operating modes, and the choice matters for both learning curve
and bill:

| | Autopilot | Standard |
|---|---|---|
| Node management | Google manages nodes; you never see a VM | You provision and manage node pools yourself |
| Billing unit | Per-pod CPU/memory/storage requests | Per-node, whether pods are using the capacity or not |
| Flexibility | Restricted to Google-managed configurations | Full control (custom machine types, DaemonSets, GPUs, taints) |
| Good for | Learning, most application workloads | Workloads needing node-level customization |

This module uses **Autopilot** — it bills only for what your pods actually
request, so an idle learning cluster costs far less than a Standard cluster
sitting on always-on nodes.

## Enable the API and create a cluster

```bash
gcloud services enable container.googleapis.com

gcloud container clusters create-auto gcp-mastery-cluster \
  --region=us-central1
```

Cluster creation takes several minutes. Once it's ready:

```bash
gcloud container clusters list
# NAME                 LOCATION      STATUS   NUM_NODES
# gcp-mastery-cluster   us-central1  RUNNING  3
```

`NUM_NODES` on Autopilot is Google-managed and will change on its own as pods
are scheduled — don't expect it to match anything you configured explicitly.

## Get kubectl talking to the cluster

`gcloud` doesn't run `kubectl` commands directly — it writes cluster
credentials into your local kubeconfig so the standalone `kubectl` binary can
reach the cluster:

```bash
gcloud components install kubectl   # if kubectl isn't already installed

gcloud container clusters get-credentials gcp-mastery-cluster \
  --region=us-central1

kubectl cluster-info
kubectl get nodes
```

## Deploy your first workload

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-web
  template:
    metadata:
      labels:
        app: hello-web
    spec:
      containers:
        - name: hello-web
          image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods
# NAME                         READY   STATUS    RESTARTS   AGE
# hello-web-6d9f8b7c5f-4x2p9   1/1     Running   0          40s
# hello-web-6d9f8b7c5f-r7k2m   1/1     Running   0          40s
```

On Autopilot, the `resources.requests` block isn't optional the way it is on
Standard — Autopilot uses it both to size the underlying (invisible) node
capacity and to compute your bill, and will reject pods that omit it entirely
for CPU/memory.

## Expose it with a Service

A Deployment alone has no stable network identity. A `LoadBalancer` Service
provisions an external GCP load balancer in front of your pods:

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-web-svc
spec:
  type: LoadBalancer
  selector:
    app: hello-web
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f service.yaml
kubectl get service hello-web-svc
# NAME            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
# hello-web-svc   LoadBalancer   10.24.3.112    34.72.xxx.xxx   80:31894/TCP   90s

curl http://34.72.xxx.xxx
# Hello, world!
# Version: 2.0.0
# Hostname: hello-web-6d9f8b7c5f-4x2p9
```

`EXTERNAL-IP` shows `<pending>` for the first minute or two while GCP
provisions the backing load balancer — this is normal, not a failure.

## Scaling and updating

```bash
# Scale the Deployment manually
kubectl scale deployment hello-web --replicas=4

# Roll out a new image version (zero downtime, one pod at a time)
kubectl set image deployment/hello-web hello-web=us-docker.pkg.dev/google-samples/containers/gke/hello-app:2.1

kubectl rollout status deployment/hello-web
kubectl rollout undo deployment/hello-web   # roll back if something's wrong
```

Autopilot also supports the Horizontal Pod Autoscaler, which reacts to load
instead of a fixed replica count — the mechanism Module 02 covers in depth
for VM-based workloads applies conceptually here too:

```bash
kubectl autoscale deployment hello-web --min=2 --max=6 --cpu-percent=60
kubectl get hpa
```

## Gotchas

- **IAM, not just Kubernetes RBAC.** Your Google identity needs a GKE IAM
  role (`roles/container.developer` at minimum, `roles/container.admin` to
  manage clusters) *before* Kubernetes RBAC is even relevant — a common
  first-timer surprise is a perfectly valid `kubectl` command failing with a
  403 because the underlying GCP identity lacks the IAM role, not because of
  anything in the YAML.
- **Regional clusters span 3 zones by default.** A `--region=us-central1`
  Autopilot cluster spreads control-plane replicas and node capacity across
  three zones for resilience — this is good for uptime, but the effective
  cost/quota surface is spread across all three, which can surprise you when
  reading regional CPU quota errors.
- **Workload Identity is how pods get GCP permissions.** Never mount a
  service account JSON key into a pod. Bind a Kubernetes service account to a
  GCP IAM service account via Workload Identity (`gcloud iam
  service-accounts add-iam-policy-binding ... --role=roles/iam.workloadIdentityUser`)
  so pods authenticate to GCP APIs without any key material on disk at all.
- **LoadBalancer Services are not free.** Each one provisions a real GCP
  forwarding rule and external IP that bill continuously — for many small
  services behind one cluster, an Ingress (using the load-balancing concepts
  from [Module 02](02-autoscaling-load-balancing.md)) that fans out to
  multiple Services on one IP is cheaper than one `LoadBalancer` Service per
  app.

## Cleanup

```bash
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
gcloud container clusters delete gcp-mastery-cluster --region=us-central1 --quiet
```

Deleting the cluster removes its nodes and control plane, and the
`LoadBalancer` Service's external IP/forwarding rule along with it — but only
if you deleted the Service *before* the cluster, or explicitly clean up
orphaned forwarding rules afterward with `gcloud compute forwarding-rules
list`.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud container clusters create-auto` | Create an Autopilot cluster (Google manages nodes). |
| `gcloud container clusters get-credentials` | Write cluster credentials into local kubeconfig for `kubectl`. |
| `kubectl apply -f <file>.yaml` | Create/update resources from a manifest. |
| `kubectl get pods` / `get service` / `get nodes` | Inspect cluster state. |
| `kubectl scale deployment <name> --replicas=N` | Manually change replica count. |
| `kubectl autoscale deployment <name> --min --max --cpu-percent` | Attach a Horizontal Pod Autoscaler. |
| `kubectl set image deployment/<name> <container>=<image>` | Roll out a new image version. |
| `kubectl rollout status` / `rollout undo` | Watch or reverse a rollout. |
| `gcloud container clusters delete` | Tear down the cluster (control plane + nodes). |

## Exercise

Create an Autopilot cluster, deploy the `hello-app` Deployment and
`LoadBalancer` Service above, and confirm `curl` against the external IP
returns a response. Then scale to 4 replicas, roll out the `2.1` image tag,
watch `kubectl rollout status` until it completes, and roll it back with
`kubectl rollout undo`. Finish by deleting the Service, the Deployment, and
the cluster, and confirm with `gcloud container clusters list` that nothing
is left running.
