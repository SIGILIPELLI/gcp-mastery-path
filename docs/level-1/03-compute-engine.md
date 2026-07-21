# 03 · Compute Engine

**Compute Engine** is GCP's infrastructure-as-a-service offering: virtual
machines you fully control, billed by the second. This module creates a VM,
opens firewall access to it, connects over SSH, and cleans up afterward.

## Machine types

A machine type is a bundle of vCPUs and memory. A few families you'll see
constantly:

| Family | Shape | Good for |
|---|---|---|
| `e2` | Balanced, cost-optimized | General-purpose, dev/test, the free-tier `e2-micro` |
| `n2`/`n2d` | Balanced, higher performance | Production general workloads |
| `c2`/`c3` | Compute-optimized | CPU-bound batch/HPC work |
| `m1`/`m2` | Memory-optimized | Large in-memory databases, caches |

Names look like `e2-micro`, `e2-medium`, `n2-standard-4` (family-tier-vCPUs).
The **free tier** covers one `e2-micro` (or equivalent) instance per month in
specific US regions (`us-west1`, `us-central1`, `us-east1`) — use one of
those regions in this module to stay inside it.

## Create a VM instance

```bash
gcloud compute instances create gcp-mastery-vm \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=http-server
```

- `--image-family`/`--image-project` picks the OS image — Debian, Ubuntu,
  Container-Optimized OS, and Windows Server images are all available via
  `gcloud compute images list`.
- `--tags` attaches network tags used to target firewall rules (below).

Check on it:

```bash
gcloud compute instances list
# NAME             ZONE           MACHINE_TYPE  STATUS
# gcp-mastery-vm   us-central1-a  e2-micro      RUNNING

gcloud compute instances describe gcp-mastery-vm --zone=us-central1-a
```

## Firewall rules

GCP VPCs are **deny-by-default** for ingress. A brand-new VM has no inbound
access at all except what an explicit firewall rule allows. Every default
network ships with implied allow rules for internal traffic and SSH via
`35.235.240.0/20` (Identity-Aware Proxy range) is **not** open by default —
you still need a rule for direct SSH on port 22.

```bash
# Allow SSH from anywhere (fine for learning; scope to your IP in real use)
gcloud compute firewall-rules create allow-ssh \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=0.0.0.0/0

# Allow HTTP only to instances tagged http-server
gcloud compute firewall-rules create allow-http \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server
```

Network tags (`--target-tags`) let one firewall rule apply narrowly to only
the instances that need it, instead of every VM on the network.

```bash
gcloud compute firewall-rules list
```

## SSH access

`gcloud` handles key generation and upload for you — no manual `ssh-keygen`
or console copy-pasting required:

```bash
gcloud compute ssh gcp-mastery-vm --zone=us-central1-a
```

The first run generates an SSH keypair (if you don't have one), pushes the
public key to project/instance metadata, and opens a session — all in one
command. Once connected, treat it like any Linux box:

```bash
sudo apt update && sudo apt install -y nginx
sudo systemctl status nginx
```

Combined with the `allow-http` rule and the instance's external IP
(`gcloud compute instances describe gcp-mastery-vm --zone=us-central1-a
--format="get(networkInterfaces[0].accessConfigs[0].natIP)"`), you now have a
web page reachable from the public internet.

## Startup scripts

Bake setup into the instance so it configures itself on boot, instead of
SSH-ing in by hand every time:

```bash
gcloud compute instances create gcp-mastery-vm-2 \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=http-server \
  --metadata=startup-script='#! /bin/bash
apt-get update
apt-get install -y nginx
echo "Hello from Compute Engine" > /var/www/html/index.html'
```

## Stopping vs. deleting

| Action | Compute billing | Disk billing | Static IP billing |
|---|---|---|---|
| `stop` | Stops | Continues | Continues (if reserved) |
| `delete` | Stops | Stops (unless disk kept) | Continues if not released |

```bash
# Stop (keeps the disk, no compute charges, can restart later)
gcloud compute instances stop gcp-mastery-vm --zone=us-central1-a

# Delete entirely (also deletes attached disks by default)
gcloud compute instances delete gcp-mastery-vm --zone=us-central1-a --quiet
```

## Cleanup

```bash
gcloud compute instances delete gcp-mastery-vm gcp-mastery-vm-2 --zone=us-central1-a --quiet
gcloud compute firewall-rules delete allow-ssh allow-http --quiet
```

## Cheat sheet

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

## Exercise

Create an `e2-micro` VM in `us-central1-a` tagged `http-server`, with a
startup script that installs nginx and writes a custom `index.html`. Add a
firewall rule allowing port 80 to `http-server`-tagged instances, then curl
the VM's external IP from your machine to confirm it serves your page. When
done, delete the VM and the firewall rule.
