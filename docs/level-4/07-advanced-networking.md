# 07 · Advanced Networking (Interconnect, Network Connectivity Center)

Level 3 introduced Interconnect for single hybrid links. At scale, orgs
need many sites (multiple data centers, multiple VPCs, multiple clouds)
connected without a hand-rolled mesh of point-to-point tunnels — that's
what **Network Connectivity Center (NCC)** is for.

## The hub-and-spoke problem Peering can't solve

Recall from Level 3: VPC Peering isn't transitive. Connecting 5 VPCs
pairwise needs 10 peering relationships, and adding a 6th needs 5 more.
NCC solves this with a **hub** that spokes attach to, giving every spoke
reachability to every other spoke through the hub, without pairwise
peering.

```bash
gcloud network-connectivity hubs create global-hub \
  --description="Central connectivity hub"

gcloud network-connectivity spokes linked-vpc-network create vpc-a-spoke \
  --hub=global-hub \
  --vpc-network=vpc-a \
  --region=us-central1

gcloud network-connectivity spokes linked-vpc-network create vpc-b-spoke \
  --hub=global-hub \
  --vpc-network=vpc-b \
  --region=us-central1
```

```bash
gcloud network-connectivity hubs list-spokes global-hub
# NAME            TYPE                LOCATION
# vpc-a-spoke     VPC_NETWORK         us-central1
# vpc-b-spoke     VPC_NETWORK         us-central1
```

Resources in `vpc-a` and `vpc-b` can now reach each other through the hub
— and a third VPC added later just needs one more spoke, not N more
peering relationships.

**Gotcha — NCC's VPC spokes have their own transitivity limits.** VPC
spokes attached to a hub *don't* automatically get transitive connectivity
to Hybrid spokes (on-prem via Interconnect/VPN) unless the hub is
explicitly configured for that — check `gcloud network-connectivity hubs
describe global-hub` for the routing mode (`PRESET` vs `CUSTOM`) rather
than assuming every spoke type talks to every other type by default.

## Interconnect for hybrid spokes at scale

```bash
gcloud network-connectivity spokes linked-interconnect-attachments create dc1-spoke \
  --hub=global-hub \
  --region=us-central1 \
  --interconnect-attachments=projects/my-project/regions/us-central1/interconnectAttachments/dc1-attachment
```

Now an on-prem data center connected via Interconnect is a spoke on the
same hub as the VPC spokes — a single hub becomes the org's one place to
reason about "what can reach what," instead of tracing individual peering
and VPN configs per pair.

## Private Service Connect (PSC)

For consuming managed services (yours or a vendor's) privately, without
exposing them to the public internet or requiring VPC peering with the
producer:

```bash
gcloud compute addresses create psc-endpoint-ip \
  --region=us-central1 \
  --subnet=app-subnet

gcloud compute forwarding-rules create psc-to-vendor-api \
  --region=us-central1 \
  --network=app-vpc \
  --address=psc-endpoint-ip \
  --target-service-attachment=projects/vendor-project/regions/us-central1/serviceAttachments/vendor-api
```

```bash
gcloud compute forwarding-rules describe psc-to-vendor-api --region=us-central1 --format="value(IPAddress,pscConnectionStatus)"
# 10.10.5.20  ACCEPTED
```

Traffic to `10.10.5.20` from inside `app-vpc` now reaches the vendor's
service over Google's internal network — no public IP, no VPC peering with
the vendor's project required, and the vendor controls who can connect via
their service attachment's accept list.

**Gotcha — PSC connection status can sit at `PENDING` indefinitely.** The
producer side must explicitly accept the consumer's project/network in
their service attachment's connection policy; a `PENDING` status that
never becomes `ACCEPTED` almost always means the producer hasn't approved
your project yet — this is a coordination step outside your own gcloud
commands.

## Cloud NAT at scale, and its port-exhaustion gotcha

```bash
gcloud compute routers nats create prod-nat \
  --router=prod-router \
  --region=us-central1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

```bash
gcloud compute routers get-nat-mapping-info prod-router --region=us-central1
# instanceName    natIpPortRanges          numTotalDrainNatPorts
# web-01          34.72.10.5:10000-10063   0
```

**Gotcha — auto-allocated NAT IPs give each VM a fixed port range by
default (64 ports), which is easy to exhaust** for a VM making many
outbound connections (e.g., high-throughput calls to external APIs).
Symptoms are silent connection failures, not an obvious quota error.
Increase `--min-ports-per-vm` explicitly for connection-heavy workloads:

```bash
gcloud compute routers nats update prod-nat \
  --router=prod-router --region=us-central1 \
  --min-ports-per-vm=2048
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud network-connectivity hubs create` | Create an NCC hub for non-transitive-peering-free mesh connectivity. |
| `spokes linked-vpc-network create` | Attach a VPC as a spoke to the hub. |
| `spokes linked-interconnect-attachments create` | Attach an on-prem Interconnect link as a hub spoke. |
| `gcloud compute forwarding-rules create --target-service-attachment=` | Consume a service privately via PSC. |
| `gcloud compute routers nats update --min-ports-per-vm=` | Avoid NAT port exhaustion for connection-heavy VMs. |

## Exercise

Design an NCC hub connecting three VPCs (no pairwise peering) and describe
how many peering relationships this would have required without NCC as a
4th and 5th VPC are added. Then write the PSC forwarding-rule command to
privately consume a hypothetical vendor API, and describe what `PENDING`
vs `ACCEPTED` connection status means and who controls the transition.
