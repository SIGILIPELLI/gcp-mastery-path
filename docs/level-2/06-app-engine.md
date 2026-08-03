# 06 · App Engine

Level 1 deployed containers with Cloud Run and VMs with Compute Engine.
**App Engine** predates both — it's Google's original fully-managed
Platform-as-a-Service: push code, and GCP handles the runtime, scaling, and
load balancing for you, with a narrower but even simpler deployment model
than Cloud Run.

## Standard vs. Flexible environment

| | Standard | Flexible |
|---|---|---|
| Runtime | Sandboxed language runtimes (Python, Node, Go, Java, etc.) | Any Docker container |
| Cold start | Milliseconds — instances scale to zero | Minutes — runs on Compute Engine VMs under the hood |
| Scale to zero | Yes | No — always at least one instance running |
| Custom binaries/native deps | Restricted | Fully supported |
| Best for | Stateless HTTP APIs, low-traffic apps | Workloads needing custom runtime dependencies |

This module uses **Standard** — it's the cheaper, faster-scaling default for
the common case (a web app with no unusual system dependencies).

## Enable the API and initialize

```bash
gcloud services enable appengine.googleapis.com

gcloud app create --region=us-central
```

An App Engine **application** is a one-time, per-project, permanent choice —
once created, its region cannot be changed, and a project can have at most
one App Engine application (though it can host many services within it).

## The app.yaml

```yaml
# app.yaml
runtime: python312

entrypoint: gunicorn -b :$PORT main:app

instance_class: F1

automatic_scaling:
  min_instances: 0
  max_instances: 5
  target_cpu_utilization: 0.65

env_variables:
  ENVIRONMENT: "production"
```

```python
# main.py
from flask import Flask
app = Flask(__name__)

@app.route("/")
def index():
    return "Hello from App Engine Standard!"
```

## Deploy

```bash
gcloud app deploy app.yaml --quiet
```

```text
Services to deploy:

descriptor:      [app.yaml]
source:           [/home/user/app]
target project:   [gcp-mastery-path-123]
target service:   [default]
target version:   [20260803t101500]
target url:       [https://gcp-mastery-path-123.uc.r.appspot.com]

Beginning deployment of service [default]...
Updating service [default]...done.
Deployed service [default] to [https://gcp-mastery-path-123.uc.r.appspot.com]
```

```bash
gcloud app browse   # opens the deployed URL

curl https://gcp-mastery-path-123.uc.r.appspot.com
# Hello from App Engine Standard!
```

## Versions and traffic splitting

Every `gcloud app deploy` creates a **new version** without touching
existing ones — nothing is overwritten by default:

```bash
gcloud app versions list
# SERVICE  VERSION.ID        TRAFFIC_SPLIT  LAST_DEPLOYED
# default  20260803t101500   1.00           2026-08-03T10:15:00Z
```

Deploy a second version, then split traffic between them for a canary
rollout instead of an all-at-once cutover:

```bash
gcloud app deploy app.yaml --version=v2-canary --no-promote

gcloud app services set-traffic default \
  --splits=20260803t101500=0.9,v2-canary=0.1
```

```bash
gcloud app versions list
# SERVICE  VERSION.ID        TRAFFIC_SPLIT
# default  20260803t101500   0.90
# default  v2-canary         0.10
```

`--no-promote` deploys the version without sending it any traffic
automatically — without that flag, `gcloud app deploy` promotes the new
version to 100% traffic immediately, which is rarely what you want for a
production canary.

## Multiple services

App Engine supports multiple **services** (formerly "modules") inside one
application, each with its own `app.yaml` and independent scaling/versions:

```yaml
# api-service/app.yaml
service: api
runtime: python312
automatic_scaling:
  min_instances: 1
  max_instances: 10
```

```bash
gcloud app deploy api-service/app.yaml
gcloud app services list
# SERVICE  NUM_VERSIONS
# default  2
# api      1
```

## Cron jobs

```yaml
# cron.yaml
cron:
  - description: "Daily report email"
    url: /tasks/daily-report
    schedule: every 24 hours
    timezone: America/Chicago
```

```bash
gcloud app deploy cron.yaml
gcloud app services describe default   # confirm deploy
```

App Engine cron calls your own app's URL on schedule with a special header
(`X-Appengine-Cron: true`) — your route handler should verify that header (or
restrict the route to internal traffic) so the endpoint can't be triggered by
an arbitrary public request.

## Gotchas

- **The region choice is permanent.** `gcloud app create --region=` cannot
  be changed later without deleting and recreating the entire application
  (and every service/version in it) — pick deliberately.
- **`min_instances: 0` means real cold starts.** Standard environment cold
  starts are fast (milliseconds to low seconds) but not zero — a
  latency-sensitive API should set `min_instances: 1+` and accept the small
  always-on cost instead.
- **`--no-promote` is easy to forget.** Omitting it sends 100% of traffic to
  the brand-new version immediately — always use it for anything you intend
  to canary or manually verify first.
- **Old versions keep costing money/quota** until explicitly deleted —
  `gcloud app deploy` never removes previous versions automatically, so a
  team that deploys often accumulates dozens of dormant versions.
- **Flexible environment has no scale-to-zero,** so it bills continuously
  even at rest — don't reach for it unless Standard's sandboxed runtimes
  genuinely can't support what you need.

## Cleanup

```bash
gcloud app services set-traffic default --splits=20260803t101500=1.0

gcloud app versions delete v2-canary --service=default --quiet
gcloud app versions delete 20260803t101500 --service=default --quiet
gcloud app services delete api --quiet
```

An App Engine application itself cannot be deleted independently of the GCP
project — the practical cleanup for a learning project is deleting unused
versions/services, or deleting the whole project when you're done with it
entirely.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud app create --region=` | One-time creation of the App Engine application. |
| `gcloud app deploy app.yaml` | Deploy a new version (promotes to 100% traffic by default). |
| `gcloud app deploy --version= --no-promote` | Deploy without shifting traffic (for canaries). |
| `gcloud app services set-traffic --splits=` | Split traffic between versions by percentage. |
| `gcloud app versions list` / `delete` | Inspect and remove old versions. |
| `gcloud app services list` / `delete` | Inspect and remove services. |
| `gcloud app deploy cron.yaml` | Deploy scheduled tasks. |

## Exercise

Deploy a minimal Flask (or equivalent) app to App Engine Standard, confirm it
responds via `gcloud app browse`. Make a small change, deploy it as
`--version=v2-canary --no-promote`, then split traffic 90/10 between the two
versions and confirm with repeated `curl` calls that roughly 1 in 10 requests
hits the new version. Finish by promoting `v2-canary` to 100%, then delete
the older version.
