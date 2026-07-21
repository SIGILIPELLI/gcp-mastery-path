# 05 · VPC Networking Basics

A **VPC (Virtual Private Cloud)** is your private network inside GCP —
global in scope, with subnets carved out region by region. This module
covers creating a custom VPC, subnets, firewall rules, and routes.

## The default network

Every new project comes with a **default** auto-mode VPC: one subnet per
region, pre-created, with permissive default firewall rules (internal
traffic and SSH-from-IAP allowed). It's fine for quick experiments — that's
what the Compute Engine module used — but production setups almost always
use a **custom-mode VPC** where you control exactly which subnets exist and
where.

```bash
gcloud compute networks list
gcloud compute networks subnets list --filter="network:default"
```

## Creating a custom VPC and subnets

```bash
# A VPC with no auto-created subnets
gcloud compute networks create app-vpc --subnet-mode=custom

# A subnet in one region
gcloud compute networks subnets create app-subnet-central \
  --network=app-vpc \
  --region=us-central1 \
  --range=10.0.0.0/24

# A second subnet in another region -- the VPC spans regions, subnets don't
gcloud compute networks subnets create app-subnet-east \
  --network=app-vpc \
  --region=us-east1 \
  --range=10.0.1.0/24
```

Key idea: a **VPC is global**, a **subnet is regional**. Resources in
different subnets of the same VPC can reach each other over Google's
internal backbone without ever touching the public internet, subject to
firewall rules.

## Firewall rules revisited

Firewall rules are attached to a VPC (not a subnet), and evaluated per
instance based on network tags, service accounts, or the entire network.
Rules have a **priority** (lower number = evaluated first, default 1000) and
a **direction** (ingress/egress).

```bash
# Allow internal traffic between anything in this VPC's subnet ranges
gcloud compute firewall-rules create app-vpc-allow-internal \
  --network=app-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp,udp,icmp \
  --source-ranges=10.0.0.0/8

# Allow SSH only from a specific admin IP, at higher priority than a
# lower-priority broad deny
gcloud compute firewall-rules create app-vpc-allow-ssh-admin \
  --network=app-vpc \
  --direction=INGRESS \
  --priority=900 \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=203.0.113.10/32
```

```bash
gcloud compute firewall-rules list --filter="network:app-vpc"
```

Because custom-mode VPCs start with **zero** implied allow rules beyond the
built-in internal-and-IAP defaults, you must explicitly allow anything you
need — this is the deny-by-default posture that makes custom VPCs the
recommended choice once you're past pure experimentation.

## Routes

Every VPC gets an implicit **default route** (`0.0.0.0/0` via the internet
gateway) so instances with an external IP can reach the internet, plus
implicit **subnet routes** so instances in different subnets of the same VPC
can reach each other. You add custom routes for things like routing specific
traffic through a NAT gateway, a VPN tunnel, or an appliance VM.

```bash
gcloud compute routes list --filter="network:app-vpc"

# Example: send traffic to a specific external range through a
# next-hop instance (e.g. a NAT or proxy VM)
gcloud compute routes create app-vpc-custom-route \
  --network=app-vpc \
  --destination-range=192.0.2.0/24 \
  --next-hop-instance=nat-gateway-vm \
  --next-hop-instance-zone=us-central1-a \
  --priority=1000
```

## Private instances and Cloud NAT

A VM with **no external IP** can't be reached from, or reach, the public
internet directly — good for security, but it also can't `apt-get update`.
**Cloud NAT** gives private instances outbound-only internet access without
giving them a public IP:

```bash
gcloud compute routers create app-vpc-router \
  --network=app-vpc \
  --region=us-central1

gcloud compute routers nats create app-vpc-nat \
  --router=app-vpc-router \
  --region=us-central1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

## Launching a VM inside the custom VPC

```bash
gcloud compute instances create app-vm-1 \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --network=app-vpc \
  --subnet=app-subnet-central \
  --no-address
```

`--no-address` skips the external IP entirely — this VM relies on Cloud NAT
for outbound access and is unreachable directly from the internet.

## Cleanup

```bash
gcloud compute instances delete app-vm-1 --zone=us-central1-a --quiet
gcloud compute routers nats delete app-vpc-nat --router=app-vpc-router --region=us-central1 --quiet
gcloud compute routers delete app-vpc-router --region=us-central1 --quiet
gcloud compute routes delete app-vpc-custom-route --quiet
gcloud compute firewall-rules delete app-vpc-allow-internal app-vpc-allow-ssh-admin --quiet
gcloud compute networks subnets delete app-subnet-central --region=us-central1 --quiet
gcloud compute networks subnets delete app-subnet-east --region=us-east1 --quiet
gcloud compute networks delete app-vpc --quiet
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud compute networks create --subnet-mode=custom` | Create a VPC with no auto-created subnets. |
| `gcloud compute networks subnets create` | Create a regional subnet inside a VPC. |
| `gcloud compute firewall-rules create` | Allow/deny traffic by tag, service account, or range. |
| `gcloud compute routes list/create` | View or add routes for a VPC. |
| `gcloud compute routers create` / `nats create` | Set up Cloud NAT for outbound-only internet access. |
| `--no-address` (on `instances create`) | Launch a VM with no external IP. |
| `gcloud compute networks delete` | Delete a VPC (must delete its subnets/rules first). |

## Exercise

Create a custom-mode VPC with one subnet, a firewall rule allowing internal
traffic only, and a private (`--no-address`) VM inside it. Set up Cloud NAT
so the VM can still reach the internet, SSH in via IAP
(`gcloud compute ssh app-vm-1 --zone=us-central1-a --tunnel-through-iap`),
and confirm `curl -I https://cloud.google.com` succeeds from inside the VM
despite it having no public IP. Then tear everything down in the order shown
above.
