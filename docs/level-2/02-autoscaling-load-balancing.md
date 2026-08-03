# 02 · Autoscaling & Load Balancing

A single Compute Engine VM (Level 1, Module 03) has a ceiling: one machine,
one failure domain. This module builds the pattern that removes both limits —
a **managed instance group (MIG)** running identical VMs from a template,
an **autoscaler** that changes how many exist based on load, and an
**HTTP(S) load balancer** that spreads traffic across whichever VMs are
currently healthy.

## The pieces, in order

```
Instance template  →  Managed Instance Group  →  Autoscaler
                              │
                       Health check
                              │
                       Backend service  ←  URL map  ←  Target proxy  ←  Forwarding rule (+ IP)
```

Each layer is a separate `gcloud` resource. It looks like a lot of ceremony
for "run a web server," and it is — but every layer is also independently
useful (the health check alone is what lets the load balancer stop routing
to a crashed instance within seconds).

## Instance template

A template is the stamp the MIG uses to create every VM — change the
template, and new instances pick up the change, but running instances don't
until you explicitly roll them.

```bash
gcloud compute instance-templates create web-template \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=http-server \
  --metadata=startup-script='#! /bin/bash
apt-get update
apt-get install -y nginx
echo "Served by $(hostname)" > /var/www/html/index.html'
```

## Managed instance group

```bash
gcloud compute instance-groups managed create web-mig \
  --template=web-template \
  --size=2 \
  --region=us-central1
```

A **regional** MIG (`--region`) spreads its VMs across multiple zones in the
region automatically — a whole zone going down doesn't take out every
instance. A **zonal** MIG (`--zone`) is simpler but shares fate with one
zone; prefer regional for anything beyond a quick experiment.

```bash
gcloud compute instance-groups managed list-instances web-mig --region=us-central1
```

## Health check

The load balancer (and the MIG itself, for auto-healing) needs a way to know
an instance is actually serving traffic, not just powered on:

```bash
gcloud compute health-checks create http web-health-check \
  --port=80 \
  --request-path=/
```

Google's health-check probes come from a fixed range that your firewall must
explicitly allow — this trips up almost everyone the first time:

```bash
gcloud compute firewall-rules create allow-health-checks \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80 \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=http-server
```

Without this rule, the load balancer marks every backend UNHEALTHY and
returns 502s — even though `curl`-ing the instance's own external IP works
fine, because that traffic doesn't come from the health-check ranges.

## Autoscaling policy

```bash
gcloud compute instance-groups managed set-autoscaling web-mig \
  --region=us-central1 \
  --min-num-replicas=2 \
  --max-num-replicas=6 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=90
```

- `--target-cpu-utilization=0.6` means "add instances when average CPU
  crosses 60%," not a hard cap.
- `--cool-down-period` gives a newly-created instance time to finish booting
  before its (artificially low, just-started) CPU usage is counted against
  the scale-in decision.
- You can also scale on custom Cloud Monitoring metrics (queue depth,
  requests-per-second) instead of CPU with `--custom-metric-utilization`.

## Backend service, URL map, proxy, forwarding rule

```bash
# Reserve a stable public IP for the load balancer
gcloud compute addresses create web-lb-ip --global

# Backend service: the LB's view of "where does traffic go"
gcloud compute backend-services create web-backend \
  --global \
  --protocol=HTTP \
  --health-checks=web-health-check \
  --port-name=http

# Attach the MIG as a backend
gcloud compute backend-services add-backend web-backend \
  --global \
  --instance-group=web-mig \
  --instance-group-region=us-central1 \
  --balancing-mode=UTILIZATION \
  --max-utilization=0.8

# URL map: routing rules (here, everything goes to one backend)
gcloud compute url-maps create web-url-map \
  --default-service=web-backend

# Target proxy: terminates the incoming protocol (HTTP here)
gcloud compute target-http-proxies create web-http-proxy \
  --url-map=web-url-map

# Forwarding rule: binds a public IP + port to the proxy
gcloud compute forwarding-rules create web-http-rule \
  --global \
  --target-http-proxy=web-http-proxy \
  --address=web-lb-ip \
  --ports=80
```

```bash
gcloud compute addresses describe web-lb-ip --global --format='value(address)'
# 34.117.xxx.xxx

curl http://34.117.xxx.xxx
# Served by web-mig-abcd
```

Refresh a few times — you should see the hostname suffix change as the load
balancer rotates across the MIG's instances.

## Watching it scale

```bash
gcloud compute instance-groups managed describe web-mig --region=us-central1 \
  --format='value(status.autoscaler)'
```

Generate synthetic load (e.g. with `hey -z 2m -c 50 http://34.117.xxx.xxx`)
and watch `gcloud compute instance-groups managed list-instances web-mig
--region=us-central1` grow past 2 instances, then shrink back after the load
stops and the cool-down window passes.

## Regional vs. global load balancing

| | Regional external LB | Global external LB (this module) |
|---|---|---|
| Backend scope | One region | Any region, one anycast IP |
| Best for | Simple single-region apps | Multi-region, lowest-latency routing to the nearest healthy backend |
| Extra features | — | Cloud CDN, Cloud Armor attach at this layer (Modules 08) |

## Gotchas

- **Propagation isn't instant.** A newly created global forwarding rule can
  take a few minutes to become reachable everywhere — a 502 or connection
  refused in the first minute doesn't necessarily mean misconfiguration.
- **Firewall rules for health checks are the #1 support issue** for GCP load
  balancers — see above. If backends show `UNHEALTHY` in `gcloud compute
  backend-services get-health web-backend --global`, check this first.
- **`--balancing-mode=UTILIZATION` needs `--max-utilization`,** and an
  instance group with zero capacity configured silently receives zero
  traffic even while marked healthy.
- **Autoscaling reacts to averages, not instant spikes** — a sudden burst
  can outrun the time it takes new instances to boot and pass health checks.
  Set `--min-num-replicas` high enough to absorb your normal traffic without
  waiting on autoscale for the baseline.

## Cleanup

```bash
gcloud compute forwarding-rules delete web-http-rule --global --quiet
gcloud compute target-http-proxies delete web-http-proxy --quiet
gcloud compute url-maps delete web-url-map --quiet
gcloud compute backend-services delete web-backend --global --quiet
gcloud compute addresses delete web-lb-ip --global --quiet
gcloud compute instance-groups managed delete web-mig --region=us-central1 --quiet
gcloud compute instance-templates delete web-template --quiet
gcloud compute health-checks delete web-health-check --quiet
gcloud compute firewall-rules delete allow-health-checks --quiet
```

## Cheat sheet

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

## Exercise

Build the full chain above, confirm `curl` against the reserved IP round-robins
across at least 2 instances, then generate load with a tool like `hey` or
`ab` and watch `list-instances` grow. Once traffic stops and the group
scales back down, tear down every resource in the Cleanup section in order
and confirm with `gcloud compute forwarding-rules list` /
`gcloud compute instance-groups managed list` that nothing remains.
