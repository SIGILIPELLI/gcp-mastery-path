# 08 · Advanced Monitoring (Cloud Trace & Profiler)

Level 1's monitoring module covered metrics and logs. Once a system is a
handful of services calling each other, "which service is slow" stops
being answerable from logs alone — you need **distributed tracing** to see
a request's path across services, and a **continuous profiler** to see
where CPU/memory actually goes inside one service.

## Cloud Trace

Cloud Trace collects latency data per request, showing a waterfall of
spans across service boundaries. Cloud Run, App Engine, and GKE with the
right client library auto-instrument HTTP calls.

```bash
gcloud services enable cloudtrace.googleapis.com
```

Application code (Python, using OpenTelemetry, GCP's supported path since
the older Trace SDKs were deprecated):

```python
from opentelemetry import trace
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(CloudTraceSpanExporter()))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("reserve_inventory"):
    reserve_result = call_inventory_service(order_id)
```

```bash
gcloud trace traces list --project=my-project --limit=5
# TRACE_ID                          SPANS  LATENCY
# 4bf92f3577b34da6a3ce929d0e0e4736  7      842ms
```

Viewing that trace ID in the console shows each span (`reserve_inventory`,
the downstream HTTP call, `charge_payment`, etc.) nested with exact
start/duration — the waterfall immediately shows whether 800ms of the
842ms was one slow downstream call or evenly spread across all of them.

**Gotcha — sampling is on by default.** The default OpenTelemetry sampler
sends a fraction of traces, not every request, to control cost and volume.
A rare intermittent slow request may simply not get sampled and never show
up. For debugging a specific known-bad request, use `TraceIdRatioBased`
tuned up temporarily, or force-sample by propagating a trace header your
load generator controls.

**Gotcha — trace context must propagate across service calls.** If
service A calls service B over HTTP without forwarding the
`traceparent` header, Trace sees two disconnected traces instead of one
end-to-end waterfall. Most GCP client libraries and common HTTP clients
propagate this automatically when instrumented consistently — but a raw
`requests.post()` call without the OpenTelemetry HTTP instrumentation
silently breaks the chain.

## Cloud Profiler

Profiler continuously samples CPU and heap usage in production with low
overhead, aggregated across all instances of a service — answering "which
function burns the most CPU across our fleet" without attaching a debugger
to any single instance.

```python
import googlecloudprofiler

googlecloudprofiler.start(
    service="orders-api",
    service_version="1.4.2",
    verbose=3,
)
```

```bash
gcloud services enable cloudprofiler.googleapis.com
```

The console's flame graph view aggregates samples across every running
instance of `orders-api`, weighted by sample count — a function taking 40%
of flame-graph width is genuinely consuming 40% of sampled CPU time
fleet-wide, not just on one noisy instance.

**Gotcha — profiler data is fleet-aggregated, not per-request.** Profiler
answers "what's expensive in general" — it cannot tell you which specific
request triggered a given CPU spike. Pair it with Trace (per-request
latency) rather than treating either as a replacement for the other.

## Alerting on trace/latency SLOs

Combine Trace-derived latency with **Service Level Objectives**:

```bash
gcloud monitoring services create orders-api-service \
  --display-name="Orders API"

gcloud alpha monitoring slo create \
  --service=orders-api-service \
  --slo-id=latency-slo \
  --display-name="95% of requests under 500ms" \
  --goal=0.95 \
  --rolling-period=28d \
  --request-based-latency-threshold=500ms
```

```bash
gcloud alpha monitoring slo list --service=orders-api-service
# SLO_ID         GOAL   PERIOD
# latency-slo    0.95   28d
```

An SLO burns an **error budget** as the SLI (measured metric) violates the
threshold; alert on burn *rate* (e.g., "will exhaust the 28-day budget in
under 6 hours at current rate") rather than on a raw threshold breach, so
paging correlates with actual user impact instead of every brief blip.

**Gotcha — SLOs need a `Service` resource created first, and metrics
already flowing.** `gcloud monitoring services create` just registers the
logical service; if the underlying latency metric (e.g., from a Cloud Run
request-latency metric) isn't already being emitted with matching labels,
the SLO shows no data rather than erroring loudly — check `gcloud alpha
monitoring slo describe` for an empty `TimeSeries` before assuming it's
broken.

## Cheat sheet

| Command / Concept | Purpose |
|---|---|
| `gcloud services enable cloudtrace.googleapis.com` | Turn on distributed tracing collection. |
| OpenTelemetry `CloudTraceSpanExporter` | Standard instrumentation path for GCP-supported languages. |
| `gcloud trace traces list` | List recent traces and their overall latency. |
| `googlecloudprofiler.start()` | Enable continuous CPU/heap profiling for a service. |
| `gcloud monitoring services create` + `slo create` | Define an SLO and track error-budget burn. |
| Alert on burn *rate*, not raw threshold | Reduces noisy paging on brief blips. |

## Exercise

Instrument a sample Cloud Run service with OpenTelemetry tracing across two
manually-created spans (an outer request span and one inner "downstream
call" span), deploy it, generate traffic, and confirm a multi-span trace
appears via `gcloud trace traces list`. Then define an SLO for that
service at 95% of requests under 500ms over a 28-day window and describe
what an appropriate burn-rate alert threshold would be for a 1-hour vs.
6-hour detection window.
