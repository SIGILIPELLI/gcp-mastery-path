# 01 · Advanced VPC (Peering, Shared VPC, Interconnect)

Level 1 and 2 built single-project VPCs. Real organizations run dozens of
projects that still need to talk to each other, share a common network
backbone, and sometimes reach on-prem data centers. This module covers three
mechanisms for that: **VPC Peering** (direct project-to-project), **Shared
VPC** (centralized network administration), and **Cloud Interconnect**
(hybrid on-prem connectivity).

## VPC Peering

Peering connects two VPCs — in the same or different projects/orgs — so
resources communicate over Google's internal network using internal IPs,
without traversing the public internet or needing a VPN.

```bash
gcloud compute networks peerings create peer-to-prod \
  --network=dev-vpc \
  --peer-project=prod-project-id \
  --peer-network=prod-vpc \
  --auto-create-routes
```

The peering must be created from **both sides** — each project owner runs
the equivalent command pointing back at the other network. Until both halves
exist, the peering state shows `INACTIVE`.

```bash
gcloud compute networks peerings list --network=dev-vpc
# NAME              NETWORK   PEER_NETWORK  PEER_PROJECT       STATE
# peer-to-prod      dev-vpc   prod-vpc      prod-project-id    ACTIVE
```

**Gotcha — no transitivity.** If A peers with B, and B peers with C, A
cannot reach C through B. Peering is strictly pairwise. This surprises teams
building hub-and-spoke topologies with peering alone — for that, use Shared
VPC or Network Connectivity Center instead.

**Gotcha — CIDR overlap.** Peering fails outright if the two VPCs have
overlapping subnet ranges. There's no NAT-the-overlap option like some other
clouds offer; ranges must be disjoint from the start, which is why IP
planning across projects matters from day one.

## Shared VPC

Shared VPC lets one **host project** own the VPC network and subnets, while
one or more **service projects** attach their VMs, GKE clusters, etc. to
those shared subnets. Network admins in the host project control IP space
and firewall rules centrally; service-project teams just deploy workloads
into subnets they've been granted access to.

```bash
# In the host project: enable it as a Shared VPC host
gcloud compute shared-vpc enable host-project-id

# Attach a service project
gcloud compute shared-vpc associated-projects add service-project-id \
  --host-project=host-project-id
```

Grant a team's service account permission to use a specific subnet:

```bash
gcloud projects add-iam-policy-binding host-project-id \
  --member="serviceAccount:app-sa@service-project-id.iam.gserviceaccount.com" \
  --role="roles/compute.networkUser" \
  --condition="expression=resource.name.endsWith('subnetworks/app-subnet'),title=app-subnet-only"
```

That `compute.networkUser` role, scoped with an IAM condition to one subnet,
is the standard pattern for restricting which team can deploy into which
subnet without granting blanket access to the whole shared network.

```bash
gcloud compute shared-vpc get-host-project service-project-id
# name: host-project-id
```

**Gotcha — one host per service project.** A service project can attach to
exactly one Shared VPC host at a time. Migrating a project to a different
host requires detaching first — which drops connectivity for anything
still using those subnets, so plan a maintenance window.

**Gotcha — quota is host-project-wide.** All service projects draw from the
host project's per-network quotas (routes, firewall rules, forwarding
rules). A noisy service project can starve others; watch `gcloud compute
networks describe` and per-project usage dashboards, not just your own
service project's view.

## Cloud Interconnect (overview)

For hybrid connectivity to on-prem data centers, Google offers two tiers:

| | Dedicated Interconnect | Partner Interconnect |
|---|---|---|
| Connection | Direct physical link into a Google colocation facility | Via a supported service provider |
| Minimum capacity | 10 Gbps per circuit | 50 Mbps, scales up |
| Setup lead time | Weeks (physical cross-connect) | Days, provider-dependent |
| Best for | Sustained high-bandwidth, latency-sensitive workloads | Lower-volume or faster time-to-provision needs |

```bash
gcloud compute interconnects attachments dedicated create prod-attachment \
  --router=onprem-router \
  --region=us-central1 \
  --interconnect=my-interconnect \
  --vlan=100
```

Both tiers terminate into a **Cloud Router**, which speaks BGP to exchange
routes between your on-prem network and the VPC — routes aren't statically
configured, they're learned dynamically, so a route table change on either
side propagates without a redeploy.

**Gotcha — Interconnect is regional infrastructure with global routing
implications.** A single attachment lives in one region, but if your Cloud
Router is configured for global dynamic routing, on-prem routes become
reachable VPC-wide, not just from the attachment's region. Decide global vs.
regional routing intentionally — global routing spreads risk further if a
misconfigured route leaks.

## Cheat sheet

| Command / Concept | Purpose |
|---|---|
| `gcloud compute networks peerings create` | Direct VPC-to-VPC connectivity (must be created both sides). |
| Shared VPC host/service project | Centralize network admin; teams deploy into shared subnets. |
| `roles/compute.networkUser` + IAM condition | Restrict a team to one specific shared subnet. |
| Dedicated Interconnect | High-bandwidth, direct physical on-prem link. |
| Partner Interconnect | Lower-minimum, provider-mediated on-prem link. |
| Cloud Router (BGP) | Dynamic route exchange for Interconnect/VPN. |

## Exercise

Create two VPCs in the same project simulating separate teams (`team-a-vpc`,
`team-b-vpc`) with non-overlapping `/24` subnets. Peer them in both
directions and confirm `ACTIVE` state with `gcloud compute networks peerings
list`. Then enable one project as a Shared VPC host, attach a subnet, and
grant `roles/compute.networkUser` scoped to that single subnet via an IAM
condition — verify with `gcloud projects get-iam-policy` that the condition
is present and correctly scoped.
