---
title: "GCP Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free GCP cheat sheet: 39 quick-reference tables from every level of the GCP Mastery Path, each linked to its full lesson."
---

# GCP Cheat Sheet

Quick-reference tables from **39 lessons** of the free GCP Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · Setup &amp; gcloud CLI](level-1/01-setup.md)

| Command | Purpose |
|---|---|
| `gcloud init` | Interactive first-time setup (account, project, region/zone). |
| `gcloud auth login` | Authenticate the CLI to a Google account. |
| `gcloud auth application-default login` | Set up ADC for client libraries and Terraform. |
| `gcloud config set project <id>` | Set the active default project. |
| `gcloud config list` | Show current CLI configuration. |
| `gcloud projects create <id>` | Create a new project from the CLI. |
| `gcloud billing projects link <id> --billing-account=<acct>` | Attach billing to a project. |
| `gcloud services enable <api>` | Turn on a GCP API for the current project. |
| `gcloud services list --enabled` | List APIs enabled on the current project. |

### [02 · IAM &amp; Security Basics](level-1/02-iam-security.md)

| Command | Purpose |
|---|---|
| `gcloud projects add-iam-policy-binding` | Grant a role to a member on a project. |
| `gcloud projects remove-iam-policy-binding` | Revoke a role from a member. |
| `gcloud projects get-iam-policy` | View the current IAM policy. |
| `gcloud iam roles list` | List available (predefined/custom) roles. |
| `gcloud iam roles describe <role>` | See exactly what permissions a role grants. |
| `gcloud iam service-accounts create` | Create a new service account. |
| `gcloud iam service-accounts keys create` | Generate a key file for a service account (use sparingly). |
| `gcloud iam service-accounts keys delete` | Revoke a service account key. |

### [03 · Compute Engine](level-1/03-compute-engine.md)

| Command | Purpose |
|---|---|
| `gcloud compute instances create` | Create a new VM. |
| `gcloud compute instances list` | List VMs and their status. |
| `gcloud compute instances describe` | Show full detail on one VM. |
| `gcloud compute instances stop/start` | Stop or start a VM (disk billing continues while stopped). |
| `gcloud compute instances delete` | Permanently delete a VM (and by default its disks). |
| `gcloud compute ssh <name>` | SSH in, auto-handling key setup. |
| `gcloud compute firewall-rules create` | Allow (or deny) traffic to matching instances. |
| `gcloud compute firewall-rules list` | List firewall rules on a network. |
| `--metadata=startup-script=...` | Run a script automatically on first boot. |

### [04 · Cloud Storage](level-1/04-cloud-storage.md)

| Command | Purpose |
|---|---|
| `gcloud storage buckets create gs://<name>` | Create a bucket. |
| `gcloud storage cp` | Upload/download/copy objects. |
| `gcloud storage rsync` | Sync a local directory and a bucket path. |
| `gcloud storage ls` | List buckets or objects. |
| `gcloud storage rm` | Delete objects (`--recursive` for a whole prefix). |
| `gcloud storage buckets update --lifecycle-file=` | Set an automated class-transition/expiry policy. |
| `gcloud storage buckets add-iam-policy-binding` | Grant access to a bucket. |
| `gcloud storage buckets update --web-main-page-suffix=` | Enable static website hosting on a bucket. |
| `gcloud storage buckets delete` | Permanently delete an (empty) bucket. |

### [05 · VPC Networking Basics](level-1/05-vpc-networking.md)

| Command | Purpose |
|---|---|
| `gcloud compute networks create --subnet-mode=custom` | Create a VPC with no auto-created subnets. |
| `gcloud compute networks subnets create` | Create a regional subnet inside a VPC. |
| `gcloud compute firewall-rules create` | Allow/deny traffic by tag, service account, or range. |
| `gcloud compute routes list/create` | View or add routes for a VPC. |
| `gcloud compute routers create` / `nats create` | Set up Cloud NAT for outbound-only internet access. |
| `--no-address` (on `instances create`) | Launch a VM with no external IP. |
| `gcloud compute networks delete` | Delete a VPC (must delete its subnets/rules first). |

### [06 · Cloud SQL](level-1/06-cloud-sql.md)

| Command | Purpose |
|---|---|
| `gcloud sql instances create` | Create a managed database instance. |
| `gcloud sql instances list/describe` | View instance status and connection info. |
| `gcloud sql databases create` | Create a logical database on an instance. |
| `gcloud sql users create` | Create a database user. |
| `gcloud sql instances patch --authorized-networks=` | Allow a specific IP for public-IP connections. |
| `cloud-sql-proxy` | Local/app-side proxy authenticating via IAM, no IP allowlisting. |
| `--add-cloudsql-instances` (Cloud Run/Functions) | Wire serverless compute to a Cloud SQL instance via Unix socket. |
| `gcloud sql backups create/list/restore` | Manage on-demand and automated backups. |
| `gcloud sql instances delete` | Permanently delete an instance and its data. |

### [07 · Cloud Functions &amp; Cloud Run](level-1/07-serverless-run-functions.md)

| Command | Purpose |
|---|---|
| `gcloud functions deploy --gen2 --trigger-http` | Deploy an HTTP-triggered function. |
| `gcloud functions deploy --trigger-bucket=` | Deploy a function triggered by Cloud Storage events. |
| `gcloud functions call` | Invoke a function directly (for non-HTTP or quick tests). |
| `gcloud run deploy --source=.` | Build (via Cloud Build) and deploy a container from source. |
| `gcloud run deploy --image=` | Deploy a pre-built container image. |
| `gcloud run services update --min-instances/--max-instances` | Tune scaling behavior. |
| `gcloud run services describe --format='value(status.url)'` | Get a service's public URL. |
| `gcloud functions delete` / `gcloud run services delete` | Tear down. |

### [08 · Cloud Monitoring &amp; Logging](level-1/08-monitoring-logging.md)

| Command | Purpose |
|---|---|
| `gcloud logging read '<filter>'` | Query logs from any resource. |
| `gcloud logging metrics create` | Turn a log filter into a chartable/alertable metric. |
| `gcloud monitoring dashboards list` | List Monitoring dashboards. |
| `gcloud monitoring time-series list` | Query raw metric data from the CLI. |
| `gcloud alpha monitoring channels create` | Create a notification channel (email, SMS, Slack, etc.). |
| `gcloud alpha monitoring policies create --policy-from-file=` | Create an alerting policy from YAML/JSON. |
| `gcloud monitoring uptime create` | Add an external uptime check for a URL. |
| `severity>=ERROR` | Log filter fragment for errors and above. |

### [09 · Infrastructure as Code (Terraform on GCP)](level-1/09-terraform-intro.md)

| Command | Purpose |
|---|---|
| `terraform init` | Download providers, set up the working directory. |
| `terraform plan` | Preview changes without applying them. |
| `terraform apply` | Create/update resources to match the config. |
| `terraform destroy` | Delete every resource the config manages. |
| `terraform output` | Print declared output values. |
| `terraform state list` | List resources tracked in the current state. |
| `resource "google_X" "name" { ... }` | Declare a GCP resource. |
| `var.name` / `variable "name" {}` | Parameterize a configuration. |

### [10 · Capstone Project](level-1/10-capstone-project.md)

| Command | Purpose |
|---|---|
| `gcloud sql instances create/delete` | Provision/remove the database — highest ongoing cost, delete first priority. |
| `gcloud run deploy --source=. --add-cloudsql-instances=` | Deploy the API wired to Cloud SQL. |
| `gcloud storage buckets create --uniform-bucket-level-access` | Create the frontend bucket. |
| `gcloud storage buckets update --web-main-page-suffix=` | Enable static website serving. |
| `gcloud run services delete` | Tear down the backend. |
| `gcloud storage buckets delete` | Tear down the frontend bucket (must be empty first). |
| `gcloud sql instances list` / `gcloud run services list` | Verify teardown left nothing running. |

## Level 2 · Intermediate

### [01 · Containers on GCP (GKE Basics)](level-2/01-gke-containers.md)

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

### [02 · Autoscaling &amp; Load Balancing](level-2/02-autoscaling-load-balancing.md)

| Command | Purpose |
|---|---|
| `gcloud compute instance-templates create` | Define the VM stamp a MIG uses. |
| `gcloud compute instance-groups managed create --region=` | Create a regional MIG from a template. |
| `gcloud compute instance-groups managed set-autoscaling` | Attach min/max replicas and a scaling signal. |
| `gcloud compute health-checks create http` | Define how the LB decides an instance is healthy. |
| `gcloud compute backend-services create/add-backend` | Define where traffic goes and attach a MIG. |
| `gcloud compute url-maps create` | Define routing rules from host/path to backend service. |
| `gcloud compute target-http-proxies create` | Terminate the incoming protocol, hand off to the URL map. |
| `gcloud compute forwarding-rules create --global` | Bind a public IP + port to the proxy. |
| `gcloud compute backend-services get-health` | Check per-instance health as seen by the LB. |

### [03 · Cloud DNS &amp; Domains](level-2/03-cloud-dns.md)

| Command | Purpose |
|---|---|
| `gcloud dns managed-zones create --dns-name=` | Create a public/private zone for a domain. |
| `gcloud dns managed-zones describe --format="value(nameServers)"` | Get the four name servers to configure at the registrar. |
| `gcloud dns record-sets transaction start/add/remove/execute` | Atomically change records in a zone. |
| `gcloud dns record-sets list` | List all records in a zone. |
| `gcloud dns managed-zones update --dnssec-state=on` | Enable DNSSEC signing. |
| `dig <name> <type> +short` | Query DNS directly to verify what's published. |

### [04 · Pub/Sub Messaging](level-2/04-pubsub-messaging.md)

| Command | Purpose |
|---|---|
| `gcloud pubsub topics create` | Create a message channel. |
| `gcloud pubsub subscriptions create --topic=` | Attach a pull subscription to a topic. |
| `gcloud pubsub subscriptions create --push-endpoint=` | Attach a push (HTTP) subscription. |
| `gcloud pubsub topics publish --message=` | Publish a message. |
| `gcloud pubsub subscriptions pull --auto-ack` | Pull and acknowledge messages manually. |
| `gcloud pubsub subscriptions update --dead-letter-topic=` | Route repeatedly-failed messages to a DLQ. |
| `gcloud pubsub subscriptions create --enable-message-ordering` | Guarantee per-ordering-key delivery order. |

### [05 · Firestore &amp; Bigtable Deep Dive](level-2/05-firestore-bigtable.md)

| Command | Purpose |
|---|---|
| `gcloud firestore databases create --type=firestore-native` | Provision the (one-time, permanent) Firestore mode. |
| `gcloud firestore export gs://<bucket>/<path>` | Back up Firestore data to Cloud Storage. |
| `gcloud firestore indexes composite create` | Declare a multi-field composite index. |
| `gcloud bigtable instances create --cluster-config=` | Provision a Bigtable instance and its first cluster. |
| `cbt createtable` / `createfamily` | Create a table and column family. |
| `cbt set` / `read` | Write/read rows via the Bigtable-specific CLI. |

### [06 · App Engine](level-2/06-app-engine.md)

| Command | Purpose |
|---|---|
| `gcloud app create --region=` | One-time creation of the App Engine application. |
| `gcloud app deploy app.yaml` | Deploy a new version (promotes to 100% traffic by default). |
| `gcloud app deploy --version= --no-promote` | Deploy without shifting traffic (for canaries). |
| `gcloud app services set-traffic --splits=` | Split traffic between versions by percentage. |
| `gcloud app versions list` / `delete` | Inspect and remove old versions. |
| `gcloud app services list` / `delete` | Inspect and remove services. |
| `gcloud app deploy cron.yaml` | Deploy scheduled tasks. |

### [07 · Secret Manager &amp; Config](level-2/07-secret-manager.md)

| Command | Purpose |
|---|---|
| `gcloud secrets create --data-file=-` | Create a secret from stdin. |
| `gcloud secrets versions add --data-file=-` | Add a new version (rotate a value). |
| `gcloud secrets versions access latest` | Read the current value. |
| `gcloud secrets versions disable <n>` | Revoke access to an old version without deleting it. |
| `gcloud secrets add-iam-policy-binding --role=roles/secretmanager.secretAccessor` | Grant read-only access to a specific identity. |
| `gcloud run deploy --set-secrets="ENV=secret:version"` | Mount a secret as an env var in Cloud Run. |
| `gcloud secrets delete` | Permanently delete a secret and all its versions. |

### [08 · Cloud CDN &amp; Cloud Armor](level-2/08-cloud-cdn-armor.md)

| Command | Purpose |
|---|---|
| `gcloud compute backend-services update --enable-cdn` | Turn on CDN caching for a backend. |
| `gcloud compute url-maps invalidate-cdn-cache --path=` | Force-expire cached objects by path. |
| `gcloud compute security-policies create` | Create a Cloud Armor policy. |
| `gcloud compute security-policies rules create <priority>` | Add an allow/deny/rate-limit rule. |
| `evaluatePreconfiguredExpr('sqli-stable')` | Use Google's built-in WAF rule sets in an expression. |
| `gcloud compute backend-services update --security-policy=` | Attach a Cloud Armor policy to a backend. |

### [09 · Cost Management &amp; Billing](level-2/09-cost-management-billing.md)

| Command | Purpose |
|---|---|
| `gcloud billing projects describe` | Confirm which billing account a project is linked to. |
| `gcloud billing budgets create --threshold-rule=percent=` | Create a budget with alert thresholds. |
| `--all-updates-rule-pubsub-topic=` | Wire budget alerts to Pub/Sub for automation. |
| `--labels=key=value` (on most `create` commands) | Tag resources for cost attribution. |
| `bq mk --dataset` | Create the BigQuery dataset billing export writes into. |
| `gcloud recommender recommendations list` | Get machine-generated cost/right-sizing suggestions. |

### [10 · Project — Scalable Web App](level-2/10-project-scalable-web-app.md)

| Command | Purpose |
|---|---|
| `gcloud container clusters create-auto` | Provision the GKE Autopilot cluster hosting both services. |
| Workload Identity binding + KSA annotation | Let pods authenticate to Firestore/Pub/Sub with no key files. |
| `gcloud pubsub subscriptions create --dead-letter-topic=` | Make the async fulfillment path failure-safe. |
| `kubectl apply -f orders-ingress.yaml` (with `ManagedCertificate`) | Provision the global LB + managed TLS cert from GKE. |
| `gcloud compute backend-services update --enable-cdn --security-policy=` | Attach CDN + Cloud Armor to the Ingress-managed backend. |
| `gcloud dns record-sets transaction ...` | Point the custom domain at the load balancer's IP. |
| `gcloud billing budgets create` | Guardrail the whole stack's spend. |

## Level 3 · Advanced

### [01 · Advanced VPC (Peering, Shared VPC, Interconnect)](level-3/01-advanced-vpc.md)

| Command / Concept | Purpose |
|---|---|
| `gcloud compute networks peerings create` | Direct VPC-to-VPC connectivity (must be created both sides). |
| Shared VPC host/service project | Centralize network admin; teams deploy into shared subnets. |
| `roles/compute.networkUser` + IAM condition | Restrict a team to one specific shared subnet. |
| Dedicated Interconnect | High-bandwidth, direct physical on-prem link. |
| Partner Interconnect | Lower-minimum, provider-mediated on-prem link. |
| Cloud Router (BGP) | Dynamic route exchange for Interconnect/VPN. |

### [02 · Kubernetes on GCP (GKE Advanced)](level-3/02-gke-advanced.md)

| Command | Purpose |
|---|---|
| `gcloud container node-pools create --spot` | Add a cost-optimized, preemptible node pool. |
| `--workload-pool=PROJECT.svc.id.goog` | Enable Workload Identity on a cluster. |
| `iam.gke.io/gcp-service-account` annotation | Bind a KSA to a GSA for pod-level auth. |
| `kubectl autoscale deployment` | Configure HPA for a deployment. |
| `gcloud container clusters update --enable-autoscaling` | Configure cluster (node) autoscaling. |
| `gcloud container fleet memberships register` | Join a cluster to a fleet for multi-cluster features. |

### [03 · Cloud Workflows &amp; Orchestration](level-3/03-workflows-orchestration.md)

| Command | Purpose |
|---|---|
| `gcloud workflows deploy` | Create/update a workflow from a YAML/JSON source file. |
| `gcloud workflows execute` | Run a workflow with input data. |
| `gcloud workflows executions describe` | Check status/result of a specific execution. |
| `try` / `retry` / `except` | Built-in per-step retry and error-handling. |
| `http.default_retry_predicate` | Standard retry-on-5xx/429 policy. |
| `roles/run.invoker` on the workflow's SA | Required to call a private Cloud Run service. |

### [04 · Advanced IAM (Org Policies, Custom Roles)](level-3/04-advanced-iam.md)

| Command | Purpose |
|---|---|
| `gcloud iam roles create` | Define a custom role with an exact permission set. |
| `gcloud org-policies set-policy` | Apply a constraint at org/folder/project scope. |
| `gcloud org-policies describe` | Check the effective policy for a constraint on a resource. |
| `gcloud iam deny-policies create` | Create a hard deny that overrides any Allow binding. |
| `--stage=TESTING` | Mark a custom role as provisional during tuning. |

### [05 · Terraform on GCP (Advanced)](level-3/05-terraform-on-gcp.md)

| Command | Purpose |
|---|---|
| `backend "gcs" {}` | Store state remotely with locking. |
| `terraform force-unlock` | Recover from a stuck state lock (verify no live apply first). |
| `module "x" { source = ... }` | Reuse a parameterized resource bundle. |
| `terraform init -upgrade` | Pick up a changed module source. |
| `terraform workspace new/select` | Separate state per environment within one backend. |
| `terraform import RESOURCE ID` | Bring an existing GCP resource under Terraform management. |

### [06 · CI/CD (Cloud Build, Artifact Registry)](level-3/06-cicd-cloud-build.md)

| Command | Purpose |
|---|---|
| `gcloud builds submit --config=` | Run a build manually from local source. |
| `gcloud builds triggers create github` | Auto-run a build config on push/PR. |
| `$SHORT_SHA`, `$PROJECT_ID` | Built-in substitutions for commit-based tagging. |
| `roles/run.admin` + `roles/iam.serviceAccountUser` on Cloud Build SA | Required for a build step to deploy to Cloud Run. |
| `gcloud artifacts repositories create` | Create a regional image/package repo. |
| `--require-approval` / `gcloud builds approve` | Manual gate before a pipeline proceeds. |

### [07 · Multi-Region &amp; Disaster Recovery](level-3/07-multi-region-disaster-recovery.md)

| Command | Purpose |
|---|---|
| `gcloud sql instances create --master-instance-name=` | Create a cross-region Cloud SQL read replica. |
| `gcloud sql instances promote-replica` | Failover: promote a replica to an independent primary (one-way). |
| `gcloud spanner instances create --config=nam3` | Multi-region Spanner with automatic synchronous failover. |
| `gcloud compute backend-services add-backend --instance-group-region=` | Add a regional backend to a global LB. |
| `gcloud compute backend-services get-health` | Check per-backend health across regions. |
| Game-day: `remove-backend` then re-add | Rehearse failover without a real outage. |

### [08 · Advanced Monitoring (Cloud Trace &amp; Profiler)](level-3/08-advanced-monitoring-trace.md)

| Command / Concept | Purpose |
|---|---|
| `gcloud services enable cloudtrace.googleapis.com` | Turn on distributed tracing collection. |
| OpenTelemetry `CloudTraceSpanExporter` | Standard instrumentation path for GCP-supported languages. |
| `gcloud trace traces list` | List recent traces and their overall latency. |
| `googlecloudprofiler.start()` | Enable continuous CPU/heap profiling for a service. |
| `gcloud monitoring services create` + `slo create` | Define an SLO and track error-budget burn. |
| Alert on burn *rate*, not raw threshold | Reduces noisy paging on brief blips. |

### [09 · Security Deep Dive (Security Command Center, VPC Service Controls)](level-3/09-security-deep-dive.md)

| Command | Purpose |
|---|---|
| `gcloud scc findings list` | List active security findings org-wide. |
| `gcloud scc settings describe` | Check which SCC tier (Standard/Premium) is active. |
| `gcloud access-context-manager perimeters create` | Wrap projects in a VPC-SC perimeter. |
| `--add-egress-policies` / `--add-ingress-policies` | Punch a scoped exception through a perimeter. |
| `perimeters dry-run enforce` | Test a perimeter's impact before enforcing it. |
| `gcloud container binauthz policy import` | Require signed/attested images at deploy time. |

### [10 · Project — Multi-Tier CI/CD Pipeline](level-3/10-project-cicd-pipeline.md)

| Command | Purpose |
|---|---|
| `gcloud compute shared-vpc enable` / `associated-projects add` | Wire a service project into a Shared VPC host. |
| `gcloud builds triggers create github` | Auto-build on push to main. |
| `gcloud workflows deploy` / `execute` | Canary-gate the actual rollout logic. |
| `sys.sleep` in a workflow | Soak period before judging canary health. |
| Workload Identity KSA/GSA binding | Pod-level read-only BigQuery access, no keys. |
| `gcloud scc findings list --filter=` | Continuous drift/misconfig detection. |

## Level 4 · Master

### [01 · GCP Architecture Framework](level-4/01-architecture-framework.md)

| Command | Purpose |
|---|---|
| `gcloud recommender recommendations list` | Pull automated cost/security/performance suggestions. |
| `gcloud recommender recommendations mark-claimed` | Record that you're acting on a recommendation. |
| `gcloud resource-manager folders create` | Build landing-zone folder structure under an org. |
| Five pillars (Ops, Security, Reliability, Cost, Performance) | The lens for every architecture review. |

### [02 · Advanced Serverless (Eventarc at Scale)](level-4/02-advanced-serverless-eventarc.md)

| Command | Purpose |
|---|---|
| `gcloud eventarc triggers create --event-filters=type=` | Wire a GCP event type to a Cloud Run destination. |
| `google.cloud.audit.log.v1.written` | React to any Cloud Audit Log entry (Admin Activity or Data Access). |
| `roles/pubsub.publisher` on GCS service agent | Required for GCS-sourced Eventarc triggers to actually fire. |
| `--enable-message-ordering` + `--dead-letter-topic` | Add ordering guarantees and a failure backstop via Pub/Sub. |
| Idempotency key by event ID/generation | Required because delivery is at-least-once. |

### [03 · Data Analytics (BigQuery, Dataflow, Dataproc)](level-4/03-data-analytics-bigquery.md)

| Command | Purpose |
|---|---|
| `PARTITION BY DATE(...)` / `CLUSTER BY` | Reduce bytes scanned, and cost, on large tables. |
| `bq query --dry_run` | Estimate bytes scanned before running a query for real. |
| `gcloud dataflow jobs list --status=active` | Catch forgotten, still-billing streaming jobs. |
| `gcloud dataflow jobs cancel` | Stop a streaming pipeline explicitly. |
| `gcloud dataproc clusters create --max-idle=` | Auto-delete an idle ephemeral Spark cluster. |
| `gcloud dataproc jobs submit pyspark` | Run a Spark job against a Dataproc cluster. |

### [04 · Kubernetes at Scale (GKE Production Patterns)](level-4/04-gke-at-scale.md)

| Command | Purpose |
|---|---|
| `--enable-dataplane-v2` | Required at cluster creation for NetworkPolicy enforcement. |
| `ResourceQuota` per namespace | Hard-cap CPU/memory/pod count per team. |
| Argo Rollouts `canary.steps` | Weighted, paused, metric-gated progressive delivery. |
| `gcloud container fleet config-management apply` | GitOps-sync manifests across a cluster fleet. |
| `--autoscaling-profile=optimize-utilization` | Trade scheduling latency for higher bin-packing efficiency. |

### [05 · Multi-Project Strategy (Organizations, Folders)](level-4/05-multi-project-strategy.md)

| Command | Purpose |
|---|---|
| `gcloud resource-manager folders create` | Build the org's folder hierarchy. |
| `gcloud projects create --folder=` | Place a new project under a specific folder. |
| `gcloud logging sinks create --include-children` | Centralize logs org-wide into one destination. |
| `gcloud billing projects link` | Attach a project to a billing account (required for billable resources). |
| `gcloud billing budgets create --filter-projects=` | Budget alert scoped to a folder's set of projects. |

### [06 · Cost Optimization at Scale](level-4/06-cost-optimization-at-scale.md)

| Command | Purpose |
|---|---|
| `gcloud compute commitments create` | Lock in a CUD for a sustained usage floor. |
| `gcloud recommender recommendations list ...MachineTypeRecommender` | Get automated rightsizing suggestions. |
| `gcloud compute disks list --filter="-users:*"` | Find unattached (orphaned) persistent disks. |
| `gcloud compute addresses list --filter="status=RESERVED"` | Find reserved IPs not attached to a resource. |
| `--labels=team=,env=,cost-center=` | Enable per-team/per-env cost attribution via billing export. |
| Billing export to BigQuery | Turn cost analysis into a repeatable SQL query. |

### [07 · Advanced Networking (Interconnect, Network Connectivity Center)](level-4/07-advanced-networking.md)

| Command | Purpose |
|---|---|
| `gcloud network-connectivity hubs create` | Create an NCC hub for non-transitive-peering-free mesh connectivity. |
| `spokes linked-vpc-network create` | Attach a VPC as a spoke to the hub. |
| `spokes linked-interconnect-attachments create` | Attach an on-prem Interconnect link as a hub spoke. |
| `gcloud compute forwarding-rules create --target-service-attachment=` | Consume a service privately via PSC. |
| `gcloud compute routers nats update --min-ports-per-vm=` | Avoid NAT port exhaustion for connection-heavy VMs. |

### [08 · Compliance &amp; Governance (Assured Workloads, Policy Intelligence)](level-4/08-compliance-governance.md)

| Command | Purpose |
|---|---|
| `gcloud assured workloads create --compliance-regime=` | Provision a regime-templated compliance folder. |
| `gcloud recommender ... google.iam.policy.Recommender` | Surface overprivileged IAM grants. |
| `gcloud asset analyze-iam-policy --full-resource-name= --permissions=` | Answer "who can do X" org-wide. |
| `gcloud org-policies set-policy gcp.resourceLocations` | Enforce data residency at org/folder scope. |
| `gsutil rewrite -k -r` | Re-encrypt existing objects after retrofitting CMEK. |

### [09 · Chaos Engineering &amp; Resilience Testing](level-4/09-chaos-engineering-resilience.md)

| Concept / Command | Purpose |
|---|---|
| Steady-state hypothesis first | Define "normal" before injecting failure. |
| `gcloud compute instance-groups managed resize --size=0` | Simulate a zonal capacity loss. |
| Istio/mesh `VirtualService` fault injection | Simulate downstream latency/errors without real outages. |
| `kubectl run ... stress` + `ResourceQuota` | Validate resource isolation actually holds under load. |
| Scheduled game days | Keep chaos testing a recurring practice, not a one-off. |

---

Go deeper with the full [GCP Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
