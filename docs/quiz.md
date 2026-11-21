---
title: "Final Quiz"
description: "Test what you have learned across all four levels of GCP Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "A VM must call Cloud Storage without any key files. What's the right approach?",
    "options": [
      "Download a JSON key",
      "Use your user",
      "Attach a least-privilege service account to the VM (keyless via the metadata server)",
      "Make bucket public"
    ],
    "correctIndex": 2,
    "explanation": "Avoid long-lived keys."
  },
  {
    "question": "A developer has roles/owner on production 'to be safe'. What's the fix?",
    "options": [
      "Keep owner",
      "Grant narrow predefined/custom roles at the smallest scope (least privilege)",
      "Use editor",
      "Share account"
    ],
    "correctIndex": 1,
    "explanation": "Primitive roles are too broad."
  },
  {
    "question": "How many usable addresses does a GCP /24 subnet have (4 reserved)?\n```python\nprint(2**8-4)\n```\n",
    "options": [
      "256",
      "254",
      "252",
      "250"
    ],
    "correctIndex": 2,
    "explanation": "The first two and last two are reserved. Verified by running it."
  },
  {
    "question": "Private VMs need to reach the internet for updates but not be reachable. What do you add?",
    "options": [
      "External IPs",
      "VPC peering",
      "Cloud VPN",
      "Cloud NAT"
    ],
    "correctIndex": 3,
    "explanation": "Outbound-only access."
  },
  {
    "question": "Static assets are slow for users worldwide. What do you add in front of the load balancer?",
    "options": [
      "Cloud NAT",
      "Cloud CDN (and appropriate Cache-Control headers)",
      "More VMs",
      "Cloud DNS"
    ],
    "correctIndex": 1,
    "explanation": "Edge caching."
  },
  {
    "question": "A Pub/Sub subscriber crashes mid-processing and the message is redelivered. What must the handler be?",
    "options": [
      "Fast",
      "Stateless only",
      "Single-threaded",
      "Idempotent: processing the same message twice must be safe"
    ],
    "correctIndex": 3,
    "explanation": "At-least-once delivery."
  },
  {
    "question": "Messages keep failing and blocking the subscription. What do you configure?",
    "options": [
      "Delete topic",
      "More subscribers",
      "Longer ack deadline",
      "A dead-letter topic with a max delivery attempts policy"
    ],
    "correctIndex": 3,
    "explanation": "Inspect and replay."
  },
  {
    "question": "Which store fits a time-series workload with millions of writes per second?",
    "options": [
      "Cloud SQL",
      "Memorystore",
      "Firestore",
      "Bigtable (design the row key to avoid hotspots)"
    ],
    "correctIndex": 3,
    "explanation": "Wide-column."
  },
  {
    "question": "Where should database passwords and API keys live?",
    "options": [
      "In env baked in image",
      "In code",
      "Secret Manager, accessed through the workload's service account",
      "In Git"
    ],
    "correctIndex": 2,
    "explanation": "Rotate and audit."
  },
  {
    "question": "A Cloud Run service has multi-second latency on the first request after idle. What reduces it?",
    "options": [
      "More CPU",
      "Set min instances (accepting the cost)",
      "VPC",
      "Bigger image"
    ],
    "correctIndex": 1,
    "explanation": "Cold starts."
  },
  {
    "question": "You need ordered steps with retries and branching across several APIs. Which service?",
    "options": [
      "Workflows",
      "Pub/Sub",
      "Eventarc",
      "Cloud Tasks"
    ],
    "correctIndex": 0,
    "explanation": "Serverless orchestration."
  },
  {
    "question": "A team's Terraform runs overwrite each other's changes. What fixes it?",
    "options": [
      "Remote state in a GCS bucket with locking",
      "Local state",
      "Manual",
      "Git state"
    ],
    "correctIndex": 0,
    "explanation": "Shared safe state."
  },
  {
    "question": "How do you let pods in GKE call Google APIs without keys?",
    "options": [
      "Node SA",
      "Workload Identity binding a Kubernetes service account to a Google service account",
      "Mount key",
      "Public"
    ],
    "correctIndex": 1,
    "explanation": "Pod-level auth."
  },
  {
    "question": "What is the maximum monthly downtime for 99.9% availability in minutes (30-day month)?\n```python\nprint(round(30*24*60*0.001,1))\n```\n",
    "options": [
      "43.2",
      "432",
      "21.6",
      "4.3"
    ],
    "correctIndex": 0,
    "explanation": "Error budget. Verified by running it."
  },
  {
    "question": "The business needs RPO of 5 minutes and RTO of 15 minutes across regions. Which design fits?",
    "options": [
      "Nightly backups",
      "Cross-region replication with automated failover (warm/active standby), tested regularly",
      "Single region",
      "Manual rebuild"
    ],
    "correctIndex": 1,
    "explanation": "Match design to objectives."
  },
  {
    "question": "Analysts' BigQuery bill is huge. What do you do first?",
    "options": [
      "Partition/cluster tables, avoid SELECT * and check bytes processed before running",
      "Buy slots",
      "More users",
      "Export"
    ],
    "correctIndex": 0,
    "explanation": "Cost follows bytes scanned."
  },
  {
    "question": "A budget alert fires but spend continues. Why?",
    "options": [
      "Bug",
      "Quota",
      "Budgets notify by default; they don't stop spending unless you automate a response",
      "Billing"
    ],
    "correctIndex": 2,
    "explanation": "Add automation."
  },
  {
    "question": "How do you stop data exfiltration from Google-managed APIs even with stolen credentials?",
    "options": [
      "Firewall",
      "Cloud Armor",
      "NAT",
      "VPC Service Controls perimeters"
    ],
    "correctIndex": 3,
    "explanation": "API-level perimeter."
  },
  {
    "question": "Only approved, signed images must run in GKE. What do you use?",
    "options": [
      "Binary Authorization",
      "Secret Manager",
      "Cloud NAT",
      "IAM"
    ],
    "correctIndex": 0,
    "explanation": "Supply chain control."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Plan non-overlapping VPC subnets

Implement plan_subnets(vpc_cidr, wanted): each subnet needs hosts + 4 reserved addresses; allocate the largest first, in the smallest aligned power-of-two block, from the VPC range, and verify there are no overlaps.

**Starter code**

```bash
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

**Expected output**

```text
batch     10.10.0.0/22     usable=1020
gke-nodes 10.10.4.0/23     usable=508
web       10.10.6.0/25     usable=124
db        10.10.6.128/27   usable=28
overlaps: False
```

<details><summary>Show solution and explanation</summary>

<pre><code>import ipaddress

RESERVED = 4        # GCP reserves 4 addresses per primary subnet range

def plan_subnets(vpc_cidr, wanted):
    &quot;&quot;&quot;wanted = [(name, hosts_needed)]. Largest first, smallest power-of-two block that fits
    hosts + RESERVED, aligned, never overlapping. Returns {name: (cidr, usable)}.&quot;&quot;&quot;
    vpc = ipaddress.ip_network(vpc_cidr)
    free = [vpc]
    plan = {}
    for name, hosts in sorted(wanted, key=lambda w: -w[1]):
        need = hosts + RESERVED
        prefix = 32
        while 2 ** (32 - prefix) &lt; need:
            prefix -= 1
        for i, block in enumerate(free):
            if block.prefixlen &lt;= prefix:
                subnet = next(block.subnets(new_prefix=prefix))
                rest = [n for n in block.address_exclude(subnet)] if subnet != block else []
                free[i:i + 1] = sorted(rest)
                plan[name] = (str(subnet), subnet.num_addresses - RESERVED)
                break
        else:
            raise ValueError(f&quot;no room for {name}&quot;)
    return plan

plan = plan_subnets(&quot;10.10.0.0/20&quot;, [(&quot;gke-nodes&quot;, 500), (&quot;web&quot;, 120), (&quot;db&quot;, 20), (&quot;batch&quot;, 1000)])
for name, (cidr, usable) in sorted(plan.items(), key=lambda kv: ipaddress.ip_network(kv[1][0])):
    print(f&quot;{name:9s} {cidr:16s} usable={usable}&quot;)
nets = [ipaddress.ip_network(c) for c, _ in plan.values()]
print(&quot;overlaps:&quot;, any(a.overlaps(b) for i, a in enumerate(nets) for b in nets[i + 1:]))
</code></pre>

<p>GCP reserves four addresses per primary range, so a /24 gives 252 usable. Planning address space up front (largest first, aligned) avoids overlaps that later block peering or hybrid connectivity. Verified by running the solution.</p>
</details>

### At-least-once delivery with dead-lettering

Implement Subscription.pull_and_handle: acknowledge handled messages, nack failures for redelivery, dead-letter after max attempts, and ignore duplicates by message id (an idempotent consumer). Simulate a poison message and a redelivered duplicate.

**Starter code**

```bash
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

**Expected output**

```text
round 1: ['m1: acked', 'm2: nack (cannot parse) -> redeliver', 'm3: acked']
round 2: ['m2: nack (cannot parse) -> redeliver']
round 3: ['m2: dead-lettered after 3 attempts']
redelivery: ['m1: duplicate ignored']
sum: 15 dead: [('m2', 'bad')]
```

<details><summary>Show solution and explanation</summary>

<pre><code>class Subscription:
    &quot;&quot;&quot;Pub/Sub-like at-least-once delivery with ack deadline, redelivery and a dead-letter topic.&quot;&quot;&quot;
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
                    log.append(f&quot;{mid}: duplicate ignored&quot;)        # idempotent consumer
                else:
                    handler(payload)
                    self.processed.add(mid)
                    log.append(f&quot;{mid}: acked&quot;)
            except Exception as e:
                if attempts &gt;= self.max_attempts:
                    self.dead.append((mid, payload))
                    log.append(f&quot;{mid}: dead-lettered after {attempts} attempts&quot;)
                else:
                    self.queue.append((mid, payload, attempts))   # nack -&gt; redeliver
                    log.append(f&quot;{mid}: nack ({e}) -&gt; redeliver&quot;)
        return log

totals = []
def handler(p):
    if p == &quot;bad&quot;:
        raise ValueError(&quot;cannot parse&quot;)
    totals.append(p)

s = Subscription()
for mid, p in [(&quot;m1&quot;, 10), (&quot;m2&quot;, &quot;bad&quot;), (&quot;m3&quot;, 5)]:
    s.publish(mid, p)
for round_no in range(1, 4):
    print(f&quot;round {round_no}:&quot;, s.pull_and_handle(handler))
s.publish(&quot;m1&quot;, 10)                    # the same message delivered again (at-least-once)
print(&quot;redelivery:&quot;, s.pull_and_handle(handler))
print(&quot;sum:&quot;, sum(totals), &quot;dead:&quot;, s.dead)
</code></pre>

<p>Pub/Sub guarantees at-least-once delivery, so the same message can arrive twice and a poison message would loop forever without a dead-letter policy. Idempotent handlers plus a max-attempts policy make that safe. This is a teaching model, not the Pub/Sub API. Verified by running the solution.</p>
</details>
