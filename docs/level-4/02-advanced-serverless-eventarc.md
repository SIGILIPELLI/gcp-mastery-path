# 02 · Advanced Serverless (Eventarc at Scale)

Level 1-2 covered Cloud Run/Functions triggered directly. **Eventarc**
generalizes triggering: any of ~150+ Google Cloud Audit Log event types, or
direct service events (GCS object finalize, Pub/Sub message, Firestore
write), can trigger a Cloud Run service or function — without the source
service knowing anything about the consumer.

## A GCS-triggered Eventarc pipeline

```bash
gcloud eventarc triggers create process-upload \
  --location=us-central1 \
  --destination-run-service=image-processor \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=uploads-bucket" \
  --service-account=eventarc-sa@my-project.iam.gserviceaccount.com
```

```bash
gcloud eventarc triggers describe process-upload --location=us-central1
# name: projects/my-project/locations/us-central1/triggers/process-upload
# eventFilters:
# - attribute: type
#   value: google.cloud.storage.object.v1.finalized
# - attribute: bucket
#   value: uploads-bucket
# destination:
#   cloudRun:
#     service: image-processor
```

Every object finalize in `uploads-bucket` now invokes `image-processor`
with a CloudEvents-formatted payload carrying the object name/bucket/
generation — the Cloud Run service reads `request.headers` for CloudEvents
metadata (`ce-type`, `ce-source`) and the body for the GCS object
resource.

**Gotcha — Eventarc for GCS requires a specific underlying transport.**
Direct GCS events route through Cloud Storage's native Pub/Sub
notification internally; the trigger's service account needs
`roles/eventarc.eventReceiver` *and* the GCS service agent needs
`roles/pubsub.publisher` on the project — a common first-deploy failure is
events silently not firing because that second binding (created
automatically by `gcloud eventarc` in most cases, but not always in
older projects) is missing:

```bash
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:service-PROJECT_NUM@gs-project-accounts.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"
```

## Audit Log-based triggers

Eventarc can also fire off **any** Cloud Audit Log entry — e.g., react to
someone calling `SetIamPolicy` on a sensitive bucket:

```bash
gcloud eventarc triggers create audit-iam-change-alert \
  --location=us-central1 \
  --destination-run-service=security-alerter \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.audit.log.v1.written" \
  --event-filters="serviceName=storage.googleapis.com" \
  --event-filters="methodName=storage.setIamPermissions" \
  --service-account=eventarc-sa@my-project.iam.gserviceaccount.com
```

This is powerful precisely because it needs no changes to the source
service — you can react to *any* API call any GCP service makes, as long
as Cloud Audit Logs capture it (Admin Activity logs are on by default and
free; Data Access logs need explicit enabling and are billed).

**Gotcha — Data Access audit logs are off by default for most services.**
An Eventarc trigger filtering on a Data Access method (e.g.,
`storage.objects.get`) will simply never fire until you enable Data Access
logging for that service in the project's audit config — and turning it on
for a busy service can meaningfully increase logging volume/cost.

## Ordering, retries, and dead-lettering

Eventarc delivers **at-least-once**, not exactly-once, and does not
guarantee ordering across events by default.

```bash
gcloud eventarc triggers create process-upload \
  --location=us-central1 \
  --destination-run-service=image-processor \
  --destination-run-region=us-central1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=uploads-bucket" \
  --service-account=eventarc-sa@my-project.iam.gserviceaccount.com \
  --event-data-content-type="application/protobuf"
```

For ordering guarantees, route through Pub/Sub with an ordering key
explicitly (Eventarc's underlying transport for many event types) rather
than relying on delivery order:

```bash
gcloud pubsub topics create image-events --message-storage-policy-allowed-regions=us-central1
gcloud pubsub subscriptions create image-events-sub \
  --topic=image-events \
  --enable-message-ordering \
  --dead-letter-topic=image-events-dlq \
  --max-delivery-attempts=5
```

**Gotcha — at-least-once means your handler must be idempotent.** A
retried event (network blip, handler timeout, non-2xx response) redelivers
the *same* event. An `image-processor` that isn't idempotent (e.g.,
appends to a counter rather than upserting by object generation) double-
processes on retry. Key all side effects by the event's unique ID or the
resource's generation number, not just "did this run."

## Eventarc vs. direct triggers vs. Workflows

| | Direct Cloud Run/Functions trigger | Eventarc | Cloud Workflows |
|---|---|---|---|
| Coupling | Source knows the exact endpoint | Source emits an event, unaware of consumers | Orchestrator explicitly calls each step |
| Fan-out | One consumer | Multiple triggers can subscribe to the same event type | N/A — sequential by design |
| Best for | Simple single-consumer pipelines | Reactive, decoupled, multi-consumer systems | Explicit multi-step business processes |

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud eventarc triggers create --event-filters=type=` | Wire a GCP event type to a Cloud Run destination. |
| `google.cloud.audit.log.v1.written` | React to any Cloud Audit Log entry (Admin Activity or Data Access). |
| `roles/pubsub.publisher` on GCS service agent | Required for GCS-sourced Eventarc triggers to actually fire. |
| `--enable-message-ordering` + `--dead-letter-topic` | Add ordering guarantees and a failure backstop via Pub/Sub. |
| Idempotency key by event ID/generation | Required because delivery is at-least-once. |

## Exercise

Design an Eventarc trigger that fires a Cloud Run service whenever a new
object lands in a GCS bucket, and a second trigger that fires a different
service whenever an IAM policy changes on that same bucket (an Audit Log
event). Write the exact IAM bindings both triggers need to actually
deliver events, and describe how you'd make the object-processing handler
idempotent against at-least-once redelivery.
