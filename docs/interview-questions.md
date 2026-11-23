---
title: "GCP Interview Questions and Answers (179 Questions)"
description: "179 GCP interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the GCP Mastery Path."
---

# GCP Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free GCP Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; gcloud CLI](level-1/01-setup.md)

**Q1.** What are the top-level containers for GCP resources?

**Answer:** Organization, folders and projects (resources live in projects)

Projects hold resources and billing links.

**Q2.** What does `gcloud auth login` do?

**Answer:** Authenticates the CLI with your Google account

Then set the project.

**Q3.** Why set a default project and region?

**Answer:** So commands target the right place without repeating flags

gcloud config set.

**Q4.** What's the free-tier caution?

**Answer:** Resources billed while they exist; delete practice resources

Budgets and alerts help.

### [02 · IAM &amp; Security Basics](level-1/02-iam-security.md)

**Q5.** What does IAM answer?

**Answer:** Who can do what on which resource

Principal + role + resource.

**Q6.** What's a service account for?

**Answer:** An identity for applications/workloads rather than people

Attach to VMs or Cloud Run.

**Q7.** What's the principle of least privilege?

**Answer:** Grant only the roles needed at the narrowest scope

Avoid primitive roles.

**Q8.** Why avoid service account keys?

**Answer:** Long-lived keys leak; use attached identities/Workload Identity

Keyless auth.

### [03 · Compute Engine](level-1/03-compute-engine.md)

**Q9.** What does Compute Engine offer?

**Answer:** Virtual machines billed per second with full OS control

IaaS.

**Q10.** What's a managed instance group?

**Answer:** Identical VMs from a template with autoscaling and auto-healing

Resilience.

**Q11.** What's a preemptible/Spot VM?

**Answer:** Cheaper VMs that can be reclaimed at any time, for fault-tolerant jobs

Cost saving.

**Q12.** How do you give a VM access to APIs?

**Answer:** Attach a service account with minimal roles

Metadata server.

### [04 · Cloud Storage](level-1/04-cloud-storage.md)

**Q13.** What are Cloud Storage classes?

**Answer:** Standard, Nearline, Coldline and Archive (by access frequency)

Cost vs retrieval.

**Q14.** What do lifecycle rules do?

**Answer:** Automatically transition or delete objects by age/conditions

Cost control.

**Q15.** What does uniform bucket-level access do?

**Answer:** Uses IAM only, disabling object ACLs

Simpler security.

**Q16.** How share an object temporarily?

**Answer:** Signed URLs

Time-limited.

### [05 · VPC Networking Basics](level-1/05-vpc-networking.md)

**Q17.** What is a GCP VPC's scope?

**Answer:** Global, with regional subnets

Unlike some clouds.

**Q18.** What do firewall rules control?

**Answer:** Allowed/denied traffic by direction, ports and targets

Implied deny ingress.

**Q19.** How many addresses in a /24 subnet, and how many are usable in GCP (4 reserved)?

```python
print(256, 256-4)
```

**Answer:** 256 252

GCP reserves the first two and last two addresses. Verified by running it.

**Q20.** What does Private Google Access do?

**Answer:** Lets VMs without external IPs reach Google APIs

Reduce exposure.

### [06 · Cloud SQL](level-1/06-cloud-sql.md)

**Q21.** What does Cloud SQL manage?

**Answer:** Patching, backups, replication and failover for MySQL/PostgreSQL/SQL Server

Managed relational DB.

**Q22.** What does high availability mean for Cloud SQL?

**Answer:** A standby in another zone with automatic failover

Zonal redundancy.

**Q23.** What's a read replica for?

**Answer:** Scaling reads and offloading analytics

Eventually consistent.

**Q24.** How should apps connect securely?

**Answer:** Cloud SQL Auth Proxy or private IP with IAM

Encrypted.

### [07 · Cloud Functions &amp; Cloud Run](level-1/07-serverless-run-functions.md)

**Q25.** When choose Cloud Run over Cloud Functions?

**Answer:** For containerized HTTP services with more control and concurrency

Both serverless.

**Q26.** What does Cloud Run scale to?

**Answer:** Zero when idle and up with traffic

Pay per use.

**Q27.** What triggers Cloud Functions?

**Answer:** Events (Pub/Sub, Storage, HTTP, etc.)

Event-driven.

**Q28.** What's a cold start?

**Answer:** Latency when a new instance starts after idle

Min instances reduce it.

### [08 · Cloud Monitoring &amp; Logging](level-1/08-monitoring-logging.md)

**Q29.** What does Cloud Logging collect?

**Answer:** Logs from GCP services and applications

Log Explorer.

**Q30.** What do Cloud Monitoring alerts use?

**Answer:** Metrics with conditions and notification channels

SLO-based.

**Q31.** What's a log-based metric?

**Answer:** A metric computed from log entries

Count errors.

**Q32.** Why define alerting policies on symptoms?

**Answer:** Users feel symptoms; alert on actionable impact

Avoid noise.

### [09 · Infrastructure as Code (Terraform on GCP)](level-1/09-terraform-intro.md)

**Q33.** Why use Terraform over console clicks?

**Answer:** Declarative, versioned, repeatable infrastructure as code

Review changes.

**Q34.** What does `terraform plan` show?

**Answer:** The changes that would be applied

Always review.

**Q35.** Where should state live for teams?

**Answer:** A remote backend like a GCS bucket with locking

Shared safe state.

**Q36.** What's drift?

**Answer:** Real infrastructure differing from the declared code

Detect with plan.

### [10 · Capstone Project](level-1/10-capstone-project.md)

**Q37.** What does the capstone combine?

**Answer:** Static frontend on Cloud Storage, Cloud Run API and Cloud SQL with IAM and monitoring

End to end.

**Q38.** How do you protect the database?

**Answer:** Private IP, least-privilege service account and no public exposure

Defense.

**Q39.** How verify the deployment?

**Answer:** Smoke tests, logs and monitoring dashboards

Observe.

**Q40.** What should you do after?

**Answer:** Destroy resources to avoid charges

terraform destroy.

## Level 2 · Intermediate

### [01 · Containers on GCP (GKE Basics)](level-2/01-gke-containers.md)

**Q41.** What does GKE Autopilot do?

**Answer:** Manages nodes for you and bills per pod resource requests

Less ops.

**Q42.** What's a Deployment?

**Answer:** A controller maintaining desired replicas of a pod template

Rolling updates.

**Q43.** What does a Service of type LoadBalancer create?

**Answer:** A Google Cloud load balancer in front of pods

External access.

**Q44.** Why set resource requests/limits?

**Answer:** Scheduling, autoscaling and cost (Autopilot bills requests)

Right-size.

### [02 · Autoscaling &amp; Load Balancing](level-2/02-autoscaling-load-balancing.md)

**Q45.** What does a global HTTP(S) load balancer give you?

**Answer:** One anycast IP routing to the nearest healthy backend, with TLS and CDN options

Layer 7.

**Q46.** What drives managed instance group autoscaling?

**Answer:** CPU utilization, load balancing capacity or custom metrics

Target utilization.

**Q47.** What do health checks do?

**Answer:** Remove unhealthy backends from rotation (and trigger auto-healing)

Correct paths and ports.

**Q48.** What's connection draining?

**Answer:** Letting in-flight requests finish before an instance is removed

Graceful.

### [03 · Cloud DNS &amp; Domains](level-2/03-cloud-dns.md)

**Q49.** What is Cloud DNS?

**Answer:** Managed authoritative DNS with global anycast

Zones and records.

**Q50.** What's a private zone?

**Answer:** DNS visible only inside selected VPC networks

Internal names.

**Q51.** What does TTL control?

**Answer:** How long resolvers cache a record

Lower before migrations.

**Q52.** Which record maps a name to an IPv4 address?

**Answer:** A record

AAAA for IPv6.

### [04 · Pub/Sub Messaging](level-2/04-pubsub-messaging.md)

**Q53.** What does Pub/Sub decouple?

**Answer:** Publishers from subscribers via topics and subscriptions

Async messaging.

**Q54.** What delivery guarantee does Pub/Sub provide by default?

**Answer:** At-least-once, so subscribers must be idempotent

Exactly-once is opt-in with limits.

**Q55.** What's a dead-letter topic?

**Answer:** Where messages go after repeated delivery failures

Inspect and replay.

**Q56.** What does ack deadline mean?

**Answer:** Time to acknowledge before redelivery

Extend for slow work.

### [05 · Firestore &amp; Bigtable Deep Dive](level-2/05-firestore-bigtable.md)

**Q57.** When choose Firestore?

**Answer:** Document data with real-time sync and flexible schema

Mobile/web apps.

**Q58.** When choose Bigtable?

**Answer:** Massive low-latency key-value/wide-column time-series workloads

Row key design matters.

**Q59.** What's the risk of a sequential Bigtable row key?

**Answer:** Hotspotting on one node

Design keys to spread load.

**Q60.** What does Firestore charge for?

**Answer:** Document reads, writes and deletes plus storage

Optimize queries.

### [06 · App Engine](level-2/06-app-engine.md)

**Q61.** What is App Engine?

**Answer:** A fully managed PaaS for web apps with automatic scaling

Standard and flexible.

**Q62.** What's the difference between standard and flexible?

**Answer:** Standard runs in sandboxes scaling to zero; flexible runs containers on VMs

Choose by needs.

**Q63.** What does traffic splitting enable?

**Answer:** Gradual rollouts across versions

Canary.

**Q64.** When prefer Cloud Run now?

**Answer:** For container-based services with more flexibility

Modern default.

### [07 · Secret Manager &amp; Config](level-2/07-secret-manager.md)

**Q65.** What does Secret Manager store?

**Answer:** Secrets like API keys and passwords with versions and IAM

Rotate and audit.

**Q66.** How do apps access secrets?

**Answer:** Via their service account with secretAccessor role

No hardcoding.

**Q67.** Why use secret versions?

**Answer:** Rotate without code changes and roll back

Pin or use latest.

**Q68.** What's logged?

**Answer:** Access to secrets in audit logs

Accountability.

### [08 · Cloud CDN &amp; Cloud Armor](level-2/08-cloud-cdn-armor.md)

**Q69.** What does Cloud CDN do?

**Answer:** Caches content at Google's edge to reduce latency and origin load

Attach to the HTTP(S) LB.

**Q70.** What does Cloud Armor provide?

**Answer:** WAF and DDoS protection with security policies

Edge defense.

**Q71.** What does a Cache-Control header affect?

**Answer:** How long content is cached at the edge

TTL.

**Q72.** What do rate-limiting rules protect against?

**Answer:** Abuse and brute force

Armor rule.

### [09 · Cost Management &amp; Billing](level-2/09-cost-management-billing.md)

**Q73.** What drives GCP cost visibility?

**Answer:** Labels, budgets and billing export to BigQuery

Allocate cost.

**Q74.** What do budgets and alerts do?

**Answer:** Notify when spend thresholds are reached (they don't stop spend by default)

Automate responses if needed.

**Q75.** What are committed use discounts?

**Answer:** Lower prices for committing to usage for 1 or 3 years

Steady workloads.

**Q76.** What's the first cleanup step?

**Answer:** Delete unused resources (idle VMs, disks, IPs, buckets)

Billed while they exist.

### [10 · Project — Scalable Web App](level-2/10-project-scalable-web-app.md)

**Q77.** What does the scalable web app project combine?

**Answer:** Load balancer, autoscaling backends, CDN, DNS and monitoring

Level 2 project.

**Q78.** How do you prove scalability?

**Answer:** Load test and watch autoscaling and latency

Evidence.

**Q79.** What makes it resilient?

**Answer:** Multiple zones, health checks and graceful draining

HA.

**Q80.** What should you monitor?

**Answer:** Latency, errors, saturation and cost

Golden signals.

## Level 3 · Advanced

### [01 · Advanced VPC (Peering, Shared VPC, Interconnect)](level-3/01-advanced-vpc.md)

**Q81.** What's Shared VPC?

**Answer:** A host project's network shared with service projects

Central network control.

**Q82.** What's VPC peering limitation?

**Answer:** Not transitive between peered networks

Use hub designs or NCC.

**Q83.** What does Cloud NAT provide?

**Answer:** Outbound internet for private instances without external IPs

Egress control.

**Q84.** What's a hybrid connectivity option?

**Answer:** Cloud VPN or Interconnect

On-prem links.

### [02 · Kubernetes on GCP (GKE Advanced)](level-3/02-gke-advanced.md)

**Q85.** What does Workload Identity do?

**Answer:** Lets pods act as Google service accounts without keys

Pod-level auth.

**Q86.** What's a node pool for?

**Answer:** Group nodes with a common configuration (machine type, labels, taints)

Workload placement.

**Q87.** What do PodDisruptionBudgets do?

**Answer:** Limit voluntary disruptions to keep availability during maintenance

Safe upgrades.

**Q88.** What's a release channel?

**Answer:** Managed upgrade cadence (Rapid/Regular/Stable)

Reduce ops.

### [03 · Cloud Workflows &amp; Orchestration](level-3/03-workflows-orchestration.md)

**Q89.** What do Workflows orchestrate?

**Answer:** Steps across services with branching, retries and error handling

Serverless orchestration.

**Q90.** When use Workflows over Pub/Sub?

**Answer:** When you need explicit ordered steps and error handling

Control flow.

**Q91.** What can a workflow step call?

**Answer:** HTTP endpoints and Google Cloud APIs

Connectors.

**Q92.** How handle failures?

**Answer:** try/except with retry policies and compensation steps

Resilience.

### [04 · Advanced IAM (Org Policies, Custom Roles)](level-3/04-advanced-iam.md)

**Q93.** What's a custom IAM role for?

**Answer:** Least-privilege permission sets not covered by predefined roles

Organization scale.

**Q94.** What are IAM conditions?

**Answer:** Restrictions like time or resource attributes on a binding

Fine-grained.

**Q95.** What does the Organization Policy service do?

**Answer:** Enforces constraints like disallowing external IPs across the org

Guardrails.

**Q96.** Why use groups for role grants?

**Answer:** Easier management than individual users

Scale.

### [05 · Terraform on GCP (Advanced)](level-3/05-terraform-on-gcp.md)

**Q97.** Why use remote state with locking?

**Answer:** Prevent concurrent applies corrupting state

GCS backend.

**Q98.** What are modules for?

**Answer:** Reusable parameterized infrastructure building blocks

DRY.

**Q99.** What does `terraform import` do?

**Answer:** Brings existing resources under Terraform management

Adopt.

**Q100.** Why separate environments with workspaces/dirs?

**Answer:** Isolate state and blast radius

Prod vs dev.

### [06 · CI/CD (Cloud Build, Artifact Registry)](level-3/06-cicd-cloud-build.md)

**Q101.** What triggers Cloud Build?

**Answer:** Git pushes/PRs via triggers or manual submits

cloudbuild.yaml.

**Q102.** What are build steps?

**Answer:** Containers executed in sequence sharing a workspace

Cloud builders.

**Q103.** Where store built images?

**Answer:** Artifact Registry

Scan for vulnerabilities.

**Q104.** What controls deploy safety?

**Answer:** Approvals, staged rollouts and automated tests

Cloud Deploy.

### [07 · Multi-Region &amp; Disaster Recovery](level-3/07-multi-region-disaster-recovery.md)

**Q105.** What's the max monthly downtime for 99.95% availability in minutes (30-day month)?

```python
print(round(30*24*60*(1-0.9995),1))
```

**Answer:** 21.6

Error budget. Verified by running it.

**Q106.** What do RTO and RPO mean?

**Answer:** Recovery time and recovery point objectives (time to restore, data loss window)

Drive DR design.

**Q107.** Which pattern gives lowest RTO?

**Answer:** Active-active across regions

Cost trade-off.

**Q108.** How test DR?

**Answer:** Regular failover drills

Prove it.

### [08 · Advanced Monitoring (Cloud Trace &amp; Profiler)](level-3/08-advanced-monitoring-trace.md)

**Q109.** What does Cloud Trace show?

**Answer:** Latency breakdown of requests across services

Distributed tracing.

**Q110.** What's an SLO?

**Answer:** A target for a service-level indicator (e.g. 99.9% requests &lt; 300 ms)

Reliability goals.

**Q111.** What's burn rate alerting?

**Answer:** Alerting on how fast the error budget is consumed

Better than thresholds.

**Q112.** What do uptime checks do?

**Answer:** Probe public endpoints from multiple regions

External view.

### [09 · Security Deep Dive (Security Command Center, VPC Service Controls)](level-3/09-security-deep-dive.md)

**Q113.** What does VPC Service Controls do?

**Answer:** Creates perimeters limiting data exfiltration from Google APIs

Defense in depth.

**Q114.** What does CMEK provide?

**Answer:** Customer-managed encryption keys in Cloud KMS

Control and audit.

**Q115.** What does Security Command Center do?

**Answer:** Aggregates findings and security posture across the org

Detection.

**Q116.** What's Binary Authorization?

**Answer:** Only allowing signed/approved container images to deploy

Supply chain.

### [10 · Project — Multi-Tier CI/CD Pipeline](level-3/10-project-cicd-pipeline.md)

**Q117.** What does the CI/CD pipeline project combine?

**Answer:** Cloud Build, Artifact Registry, automated tests and a deploy step

Level 3 project.

**Q118.** How prevent bad images from deploying?

**Answer:** Vulnerability scanning and Binary Authorization policy

Gates.

**Q119.** How roll back?

**Answer:** Redeploy the previous immutable image/revision

Cloud Run revisions.

**Q120.** What do you log?

**Answer:** Build and deploy events for audit

Traceability.

## Level 4 · Master

### [01 · GCP Architecture Framework](level-4/01-architecture-framework.md)

**Q121.** What's the Google Cloud Architecture Framework's focus?

**Answer:** Pillars like operational excellence, security, reliability, cost and performance

Design principles.

**Q122.** What's a well-architected review for?

**Answer:** Identify risks and improvements against pillars

Continuous.

**Q123.** What does 'design for failure' mean?

**Answer:** Assume components fail and build redundancy and recovery

Resilience.

**Q124.** How balance cost and reliability?

**Answer:** Match SLOs to business need rather than maximum

Trade-offs.

### [02 · Advanced Serverless (Eventarc at Scale)](level-4/02-advanced-serverless-eventarc.md)

**Q125.** What is Eventarc?

**Answer:** Routes events from Google services/SaaS to Cloud Run, Functions or Workflows

Event-driven.

**Q126.** What does idempotent event handling need?

**Answer:** Handling duplicates safely

At-least-once.

**Q127.** When use Cloud Tasks?

**Answer:** Controlled asynchronous HTTP task dispatch with rate limits and retries

Queues.

**Q128.** What does min instances do?

**Answer:** Keeps instances warm to reduce cold starts (at a cost)

Trade-off.

### [03 · Data Analytics (BigQuery, Dataflow, Dataproc)](level-4/03-data-analytics-bigquery.md)

**Q129.** What is BigQuery?

**Answer:** A serverless data warehouse billed by data processed (or slots)

Analytics.

**Q130.** How reduce BigQuery cost?

**Answer:** Partition/cluster tables and select only needed columns

Bytes scanned.

**Q131.** What does partitioning achieve?

**Answer:** Queries scan only relevant partitions

By date.

**Q132.** What are slots?

**Answer:** Units of compute capacity for BigQuery queries

Reservations.

### [04 · Kubernetes at Scale (GKE Production Patterns)](level-4/04-gke-at-scale.md)

**Q133.** How scale GKE reliably?

**Answer:** Autoscaling (HPA, cluster autoscaler), PDBs and multiple zones

Right-size.

**Q134.** What's a multi-tenant risk?

**Answer:** Noisy neighbors and security isolation; use namespaces, quotas and policies

Governance.

**Q135.** What does Config Sync/GitOps provide?

**Answer:** Declarative cluster config from Git

Consistency.

**Q136.** How limit blast radius?

**Answer:** Separate clusters/projects per environment or tenant tier

Isolation.

### [05 · Multi-Project Strategy (Organizations, Folders)](level-4/05-multi-project-strategy.md)

**Q137.** Why multiple projects?

**Answer:** Isolation of billing, IAM and quotas by environment/team

Landing zones.

**Q138.** What does the resource hierarchy enable?

**Answer:** Inherited policies and IAM from org to folder to project

Governance.

**Q139.** What's a landing zone?

**Answer:** A prepared multi-project foundation with networking, IAM and policies

Secure by default.

**Q140.** How centralize logging?

**Answer:** Aggregated log sinks to a central project/bucket

Audit.

### [06 · Cost Optimization at Scale](level-4/06-cost-optimization-at-scale.md)

**Q141.** What's the first step in cost optimization at scale?

**Answer:** Visibility: labels, billing export and per-team reports

FinOps.

**Q142.** What do rightsizing recommendations do?

**Answer:** Suggest smaller machine types based on utilization

Recommender.

**Q143.** What's a sustained use discount?

**Answer:** Automatic discount for running VMs a large part of the month (some families)

Automatic.

**Q144.** How avoid idle waste?

**Answer:** Schedule shutdowns, delete unattached disks/IPs and use autoscaling

Hygiene.

### [07 · Advanced Networking (Interconnect, Network Connectivity Center)](level-4/07-advanced-networking.md)

**Q145.** What does Network Connectivity Center do?

**Answer:** Hub-and-spoke connectivity management across VPCs and on-prem

Transitive routing.

**Q146.** What's Private Service Connect?

**Answer:** Private endpoints to Google or partner services in your VPC

Service consumption.

**Q147.** What's Premium vs Standard network tier?

**Answer:** Premium uses Google's backbone; Standard uses the public internet

Latency vs cost.

**Q148.** How secure egress?

**Answer:** Firewall rules, Cloud NAT and egress policies

Control.

### [08 · Compliance &amp; Governance (Assured Workloads, Policy Intelligence)](level-4/08-compliance-governance.md)

**Q149.** What do audit logs record?

**Answer:** Admin activity, data access and system events

Compliance.

**Q150.** What does Assured Workloads do?

**Answer:** Applies compliance controls for regulated programs

Regions and controls.

**Q151.** How enforce guardrails continuously?

**Answer:** Organization Policy constraints and policy-as-code

Prevent drift.

**Q152.** What proves compliance?

**Answer:** Evidence from logs, configs and automated checks

Audit-ready.

### [09 · Chaos Engineering &amp; Resilience Testing](level-4/09-chaos-engineering-resilience.md)

**Q153.** What is chaos engineering?

**Answer:** Injecting controlled failures to verify resilience

Hypothesis-driven.

**Q154.** What must precede chaos tests?

**Answer:** Monitoring, rollback and a limited blast radius

Safety.

**Q155.** What do game days practice?

**Answer:** Responding to failures as a team

Muscle memory.

**Q156.** What do you learn from experiments?

**Answer:** Weaknesses to fix before real incidents

Improve.

### [10 · Capstone Project — Production-Grade Cloud Architecture](level-4/10-capstone-project.md)

**Q157.** What does the capstone integrate?

**Answer:** Architecture, governance, networking, reliability and cost controls on GCP

Production platform.

**Q158.** How justify design choices?

**Answer:** Trade-offs against requirements and the framework pillars

ADRs.

**Q159.** What proves readiness?

**Answer:** DR drills, load tests, security review and cost reports

Operate.

**Q160.** What sustains the platform?

**Answer:** Ownership, automation and continuous improvement

Culture.

## Mixed questions across all levels

**Q161.** A VM must call Cloud Storage without any key files. What's the right approach?

**Answer:** Attach a least-privilege service account to the VM (keyless via the metadata server)

Avoid long-lived keys.

**Q162.** A developer has roles/owner on production 'to be safe'. What's the fix?

**Answer:** Grant narrow predefined/custom roles at the smallest scope (least privilege)

Primitive roles are too broad.

**Q163.** How many usable addresses does a GCP /24 subnet have (4 reserved)?

```python
print(2**8-4)
```

**Answer:** 252

The first two and last two are reserved. Verified by running it.

**Q164.** Private VMs need to reach the internet for updates but not be reachable. What do you add?

**Answer:** Cloud NAT

Outbound-only access.

**Q165.** Static assets are slow for users worldwide. What do you add in front of the load balancer?

**Answer:** Cloud CDN (and appropriate Cache-Control headers)

Edge caching.

**Q166.** A Pub/Sub subscriber crashes mid-processing and the message is redelivered. What must the handler be?

**Answer:** Idempotent: processing the same message twice must be safe

At-least-once delivery.

**Q167.** Messages keep failing and blocking the subscription. What do you configure?

**Answer:** A dead-letter topic with a max delivery attempts policy

Inspect and replay.

**Q168.** Which store fits a time-series workload with millions of writes per second?

**Answer:** Bigtable (design the row key to avoid hotspots)

Wide-column.

**Q169.** Where should database passwords and API keys live?

**Answer:** Secret Manager, accessed through the workload's service account

Rotate and audit.

**Q170.** A Cloud Run service has multi-second latency on the first request after idle. What reduces it?

**Answer:** Set min instances (accepting the cost)

Cold starts.

**Q171.** You need ordered steps with retries and branching across several APIs. Which service?

**Answer:** Workflows

Serverless orchestration.

**Q172.** A team's Terraform runs overwrite each other's changes. What fixes it?

**Answer:** Remote state in a GCS bucket with locking

Shared safe state.

**Q173.** How do you let pods in GKE call Google APIs without keys?

**Answer:** Workload Identity binding a Kubernetes service account to a Google service account

Pod-level auth.

**Q174.** What is the maximum monthly downtime for 99.9% availability in minutes (30-day month)?

```python
print(round(30*24*60*0.001,1))
```

**Answer:** 43.2

Error budget. Verified by running it.

**Q175.** The business needs RPO of 5 minutes and RTO of 15 minutes across regions. Which design fits?

**Answer:** Cross-region replication with automated failover (warm/active standby), tested regularly

Match design to objectives.

**Q176.** Analysts' BigQuery bill is huge. What do you do first?

**Answer:** Partition/cluster tables, avoid SELECT * and check bytes processed before running

Cost follows bytes scanned.

**Q177.** A budget alert fires but spend continues. Why?

**Answer:** Budgets notify by default; they don't stop spending unless you automate a response

Add automation.

**Q178.** How do you stop data exfiltration from Google-managed APIs even with stolen credentials?

**Answer:** VPC Service Controls perimeters

API-level perimeter.

**Q179.** Only approved, signed images must run in GKE. What do you use?

**Answer:** Binary Authorization

Supply chain control.

## Practical coding questions

### Plan non-overlapping VPC subnets

Implement plan_subnets(vpc_cidr, wanted): each subnet needs hosts + 4 reserved addresses; allocate the largest first, in the smallest aligned power-of-two block, from the VPC range, and verify there are no overlaps.

Starter code:

```
import ipaddress

RESERVED = 4        # GCP reserves 4 addresses per primary subnet range

def plan_subnets(vpc_cidr, wanted):
    # wanted = [(name, hosts_needed)]; each subnet needs hosts + RESERVED addresses. Allocate the
    # largest first, in the smallest power-of-two block that fits, aligned and non-overlapping,
    # from vpc_cidr. Return {name: (cidr_string, usable_addresses)}; raise ValueError if no room.
    return {}

plan = plan_subnets("10.10.0.0/20", [("gke-nodes", 500), ("web", 120), ("db", 20), ("batch", 1000)])
for name, (cidr, usable) in sorted(plan.items(), key=lambda kv: ipaddress.ip_network(kv[1][0])):
    print(f"{name:9s} {cidr:16s} usable={usable}")
nets = [ipaddress.ip_network(c) for c, _ in plan.values()]
print("overlaps:", any(a.overlaps(b) for i, a in enumerate(nets) for b in nets[i + 1:]))
```

Expected output:

```text
batch     10.10.0.0/22     usable=1020
gke-nodes 10.10.4.0/23     usable=508
web       10.10.6.0/25     usable=124
db        10.10.6.128/27   usable=28
overlaps: False
```

??? success "Show a solution"

    ```
    import ipaddress

    RESERVED = 4        # GCP reserves 4 addresses per primary subnet range

    def plan_subnets(vpc_cidr, wanted):
        """wanted = [(name, hosts_needed)]. Largest first, smallest power-of-two block that fits
        hosts + RESERVED, aligned, never overlapping. Returns {name: (cidr, usable)}."""
        vpc = ipaddress.ip_network(vpc_cidr)
        free = [vpc]
        plan = {}
        for name, hosts in sorted(wanted, key=lambda w: -w[1]):
            need = hosts + RESERVED
            prefix = 32
            while 2 ** (32 - prefix) < need:
                prefix -= 1
            for i, block in enumerate(free):
                if block.prefixlen <= prefix:
                    subnet = next(block.subnets(new_prefix=prefix))
                    rest = [n for n in block.address_exclude(subnet)] if subnet != block else []
                    free[i:i + 1] = sorted(rest)
                    plan[name] = (str(subnet), subnet.num_addresses - RESERVED)
                    break
            else:
                raise ValueError(f"no room for {name}")
        return plan

    plan = plan_subnets("10.10.0.0/20", [("gke-nodes", 500), ("web", 120), ("db", 20), ("batch", 1000)])
    for name, (cidr, usable) in sorted(plan.items(), key=lambda kv: ipaddress.ip_network(kv[1][0])):
        print(f"{name:9s} {cidr:16s} usable={usable}")
    nets = [ipaddress.ip_network(c) for c, _ in plan.values()]
    print("overlaps:", any(a.overlaps(b) for i, a in enumerate(nets) for b in nets[i + 1:]))
    ```

    GCP reserves four addresses per primary range, so a /24 gives 252 usable. Planning address space up front (largest first, aligned) avoids overlaps that later block peering or hybrid connectivity. Verified by running the solution.

### At-least-once delivery with dead-lettering

Implement Subscription.pull_and_handle: acknowledge handled messages, nack failures for redelivery, dead-letter after max attempts, and ignore duplicates by message id (an idempotent consumer). Simulate a poison message and a redelivered duplicate.

Starter code:

```
class Subscription:
    """Pub/Sub-like at-least-once delivery with ack deadline, redelivery and a dead-letter topic."""
    def __init__(self, max_attempts=3):
        self.max_attempts = max_attempts
        self.queue = []          # (message_id, payload, attempts)
        self.dead = []
        self.processed = set()   # handler-side idempotency (dedupe by message id)

    def publish(self, mid, payload):
        self.queue.append((mid, payload, 0))

    def pull_and_handle(self, handler):
        # one delivery round over the queued messages. For each: increment attempts; if its id was
        # already processed log "<id>: duplicate ignored" (idempotent consumer); else call handler(payload)
        # and on success add the id to processed and log "<id>: acked"; on exception, if attempts >=
        # max_attempts dead-letter it and log "<id>: dead-lettered after N attempts", else requeue it
        # (nack) and log "<id>: nack (<error>) -> redeliver". Return the list of log lines.
        return []

totals = []
def handler(p):
    if p == "bad":
        raise ValueError("cannot parse")
    totals.append(p)

s = Subscription()
for mid, p in [("m1", 10), ("m2", "bad"), ("m3", 5)]:
    s.publish(mid, p)
for round_no in range(1, 4):
    print(f"round {round_no}:", s.pull_and_handle(handler))
s.publish("m1", 10)                    # the same message delivered again (at-least-once)
print("redelivery:", s.pull_and_handle(handler))
print("sum:", sum(totals), "dead:", s.dead)
```

Expected output:

```text
round 1: ['m1: acked', 'm2: nack (cannot parse) -> redeliver', 'm3: acked']
round 2: ['m2: nack (cannot parse) -> redeliver']
round 3: ['m2: dead-lettered after 3 attempts']
redelivery: ['m1: duplicate ignored']
sum: 15 dead: [('m2', 'bad')]
```

??? success "Show a solution"

    ```
    class Subscription:
        """Pub/Sub-like at-least-once delivery with ack deadline, redelivery and a dead-letter topic."""
        def __init__(self, max_attempts=3):
            self.max_attempts = max_attempts
            self.queue = []          # (message_id, payload, attempts)
            self.dead = []
            self.processed = set()   # handler-side idempotency (dedupe by message id)

        def publish(self, mid, payload):
            self.queue.append((mid, payload, 0))

        def pull_and_handle(self, handler):
            log = []
            pending, self.queue = self.queue, []
            for mid, payload, attempts in pending:
                attempts += 1
                try:
                    if mid in self.processed:
                        log.append(f"{mid}: duplicate ignored")        # idempotent consumer
                    else:
                        handler(payload)
                        self.processed.add(mid)
                        log.append(f"{mid}: acked")
                except Exception as e:
                    if attempts >= self.max_attempts:
                        self.dead.append((mid, payload))
                        log.append(f"{mid}: dead-lettered after {attempts} attempts")
                    else:
                        self.queue.append((mid, payload, attempts))   # nack -> redeliver
                        log.append(f"{mid}: nack ({e}) -> redeliver")
            return log

    totals = []
    def handler(p):
        if p == "bad":
            raise ValueError("cannot parse")
        totals.append(p)

    s = Subscription()
    for mid, p in [("m1", 10), ("m2", "bad"), ("m3", 5)]:
        s.publish(mid, p)
    for round_no in range(1, 4):
        print(f"round {round_no}:", s.pull_and_handle(handler))
    s.publish("m1", 10)                    # the same message delivered again (at-least-once)
    print("redelivery:", s.pull_and_handle(handler))
    print("sum:", sum(totals), "dead:", s.dead)
    ```

    Pub/Sub guarantees at-least-once delivery, so the same message can arrive twice and a poison message would loop forever without a dead-letter policy. Idempotent handlers plus a max-attempts policy make that safe. This is a teaching model, not the Pub/Sub API. Verified by running the solution.

---

More free practice: the [GCP Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
