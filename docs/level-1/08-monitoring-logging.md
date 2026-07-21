# 08 · Cloud Monitoring & Logging

**Cloud Logging** collects and lets you search logs from every GCP service;
**Cloud Monitoring** collects metrics, builds dashboards, and fires alerts.
Both are on by default for most services — this module is about *using*
them, not turning them on.

## Viewing logs

Every resource you've created in this level already emits logs. Read them
without leaving the terminal:

```bash
# Recent logs from a specific Compute Engine VM
gcloud logging read \
  'resource.type="gce_instance" AND resource.labels.instance_id="INSTANCE_ID"' \
  --limit=20 --format=json

# Recent logs from a Cloud Run service
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="hello-run"' \
  --limit=20

# Only errors, across everything
gcloud logging read 'severity>=ERROR' --limit=20
```

The query language is the same one used in the Logs Explorer in the
console — filters on `resource.type`, `resource.labels.*`, `severity`,
`timestamp`, and `jsonPayload`/`textPayload` fields.

## Structured logging from your own code

Plain `print()`/`console.log()` output still reaches Cloud Logging (as
`textPayload`), but structured JSON logs are far more useful to filter and
alert on:

```python
import json
import logging

def handler(request):
    logging.info(json.dumps({
        "message": "processed request",
        "user_id": "u_123",
        "duration_ms": 42,
    }))
    return "ok"
```

On Cloud Run, Cloud Functions, and GKE, JSON written to stdout is
automatically parsed into `jsonPayload` — no logging agent configuration
needed, so `jsonPayload.user_id="u_123"` becomes a filterable field in Logs
Explorer.

## Log-based metrics

Turn a log filter into a time-series metric you can chart or alert on —
useful for things with no built-in metric, like "rate of a specific
application error message":

```bash
gcloud logging metrics create checkout-errors \
  --description="Count of checkout failures" \
  --log-filter='resource.type="cloud_run_revision" AND jsonPayload.event="checkout_failed"'
```

## Cloud Monitoring: metrics and dashboards

Every GCP resource exposes built-in metrics automatically — CPU utilization,
request count, memory, and more — visible immediately in **Metrics
Explorer** in the console, and queryable from the CLI:

```bash
gcloud monitoring dashboards list

gcloud monitoring time-series list \
  --filter='metric.type="run.googleapis.com/request_count" AND resource.labels.service_name="hello-run"' \
  --interval-start-time="$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --interval-end-time="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

## Alerting policies

An alerting policy watches a metric (or a log-based metric) against a
threshold and notifies you when it's crossed.

```bash
# Create a notification channel (email) first
gcloud alpha monitoring channels create \
  --display-name="Me" \
  --type=email \
  --channel-labels=email_address=you@example.com
```

```yaml
# high-error-rate-policy.yaml
displayName: "High Cloud Run error rate"
combiner: OR
conditions:
  - displayName: "5xx rate above threshold"
    conditionThreshold:
      filter: >
        resource.type="cloud_run_revision" AND
        metric.type="run.googleapis.com/request_count" AND
        metric.labels.response_code_class="5xx"
      comparison: COMPARISON_GT
      thresholdValue: 5
      duration: 60s
      aggregations:
        - alignmentPeriod: 60s
          perSeriesAligner: ALIGN_RATE
notificationChannels:
  - "projects/PROJECT_ID/notificationChannels/CHANNEL_ID"
```

```bash
gcloud alpha monitoring policies create --policy-from-file=high-error-rate-policy.yaml
gcloud alpha monitoring policies list
```

This fires a notification whenever the 5xx request rate on `hello-run`
exceeds 5 per minute for 60 seconds straight — the same shape you'd use for
"alert me if error rate spikes" on any service.

## Uptime checks

A simple, independent-of-your-app way to know a public endpoint is actually
reachable:

```bash
gcloud monitoring uptime create hello-run-uptime \
  --resource-type=uptime-url \
  --host="$(gcloud run services describe hello-run --region=us-central1 --format='value(status.url)' | sed 's|https://||')" \
  --path="/" \
  --protocol=https \
  --period=5
```

## Cleanup

```bash
gcloud alpha monitoring policies list --format="value(name)" | xargs -I{} gcloud alpha monitoring policies delete {} --quiet
gcloud monitoring uptime list --format="value(name)" | xargs -I{} gcloud monitoring uptime delete {} --quiet
gcloud logging metrics delete checkout-errors --quiet
```

## Cheat sheet

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

## Exercise

Redeploy the Cloud Run service from the previous module, generate a handful
of requests to it (including at least one to a route that returns a 404 or
error), then use `gcloud logging read` to find just the non-2xx requests.
Create a log-based metric counting them, and set up one alerting policy that
would notify you if that metric exceeds a small threshold. Clean up the
policy and metric afterward.
