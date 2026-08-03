# 04 · Pub/Sub Messaging

Every service so far has talked directly to another service — a client
calling Cloud Run, a load balancer calling a VM. **Pub/Sub** decouples that:
a publisher sends a message to a **topic** without knowing or caring who (if
anyone) is listening, and each **subscription** on that topic gets its own
independent copy of every message. This is the backbone pattern for
event-driven architectures — order placed, image uploaded, log line
generated — where one event can trigger multiple, unrelated downstream
actions.

## Core concepts

```
Publisher → Topic ─┬─→ Subscription A → Consumer A (e.g. send email)
                    └─→ Subscription B → Consumer B (e.g. update analytics)
```

- A **topic** is just a named message channel — it stores nothing on its own.
- A **subscription** attaches to a topic and gives each subscriber its own
  queue of undelivered messages; two subscriptions on the same topic each see
  every message independently.
- Pub/Sub guarantees **at-least-once** delivery, never *exactly once* by
  default — a consumer can see the same message twice and must handle
  duplicates (e.g. by keying processing on a message ID).

## Create a topic and subscription

```bash
gcloud services enable pubsub.googleapis.com

gcloud pubsub topics create orders

gcloud pubsub subscriptions create orders-email-sub \
  --topic=orders \
  --ack-deadline=30
```

```bash
gcloud pubsub topics list
# NAME
# projects/gcp-mastery-path-123/topics/orders

gcloud pubsub subscriptions list --format="table(name,topic)"
# NAME                    TOPIC
# orders-email-sub        projects/.../topics/orders
```

## Publish and pull messages

```bash
gcloud pubsub topics publish orders \
  --message='{"orderId": "A1001", "total": 42.50}' \
  --attribute=source=web-checkout
```

```bash
gcloud pubsub subscriptions pull orders-email-sub --auto-ack --limit=5
# ┌───────────────────────────────────────┬────────────┬──────────────────────┐
# │                DATA                   │  MESSAGE_ID │      ATTRIBUTES      │
# ├───────────────────────────────────────┼────────────┼──────────────────────┤
# │ {"orderId": "A1001", "total": 42.50}   │ 11223344   │ source=web-checkout  │
# └───────────────────────────────────────┴────────────┴──────────────────────┘
```

`--auto-ack` acknowledges the message immediately, removing it from the
subscription's backlog. Without `--auto-ack`, a message stays undelivered
(re-deliverable) until explicitly acknowledged with `gcloud pubsub
subscriptions ack` — the pattern real consumers use, so a crash mid-processing
means the message is retried rather than lost.

## Push vs. pull subscriptions

| | Pull | Push |
|---|---|---|
| How delivery works | Consumer actively calls `pull` (or uses a client library) | Pub/Sub HTTP-POSTs the message to a URL you configure |
| Good for | Workers you run continuously (GKE pods, Compute Engine) | Serverless (Cloud Run, Cloud Functions) that only run when invoked |
| Ack model | Explicit ack call | HTTP 200 response = ack; non-2xx = redelivery |

```bash
gcloud pubsub subscriptions create orders-webhook-sub \
  --topic=orders \
  --push-endpoint="https://order-processor-abcd-uc.a.run.app/pubsub" \
  --push-auth-service-account=pubsub-invoker@gcp-mastery-path-123.iam.gserviceaccount.com
```

A push subscription to a Cloud Run service normally requires the service to
allow authenticated invocations only — `--push-auth-service-account` is how
Pub/Sub attaches an identity token to each push so the target can verify the
call actually came from Pub/Sub rather than an arbitrary caller.

## Dead-letter topics

A message that repeatedly fails delivery (consumer keeps nacking or timing
out) can loop forever without a safety valve:

```bash
gcloud pubsub topics create orders-dlq

gcloud pubsub subscriptions update orders-email-sub \
  --dead-letter-topic=orders-dlq \
  --max-delivery-attempts=5
```

After 5 failed delivery attempts, the message moves to `orders-dlq` instead
of retrying indefinitely — a separate subscription on the DLQ topic lets you
inspect or replay poison messages without them clogging the main flow.

## Ordering keys

By default, Pub/Sub does not guarantee message order. For cases where order
matters per-entity (e.g. all events for one `orderId` must apply in sequence):

```bash
gcloud pubsub subscriptions create orders-ordered-sub \
  --topic=orders \
  --enable-message-ordering
```

```bash
gcloud pubsub topics publish orders \
  --message='{"orderId":"A1001","event":"shipped"}' \
  --ordering-key="A1001"
```

Ordering is only guaranteed **within one ordering key**, and only on a
subscription created with `--enable-message-ordering` — messages with
different keys (or no key at all) are still delivered independently. A
publish failure on an ordering key also pauses further messages on *that
key* until you resume it in the client library, to avoid silently
reordering around the failure.

## Terraform equivalent

```hcl
resource "google_pubsub_topic" "orders" {
  name = "orders"
}

resource "google_pubsub_topic" "orders_dlq" {
  name = "orders-dlq"
}

resource "google_pubsub_subscription" "orders_email" {
  name  = "orders-email-sub"
  topic = google_pubsub_topic.orders.name

  ack_deadline_seconds = 30

  dead_letter_policy {
    dead_letter_topic     = google_pubsub_topic.orders_dlq.id
    max_delivery_attempts = 5
  }
}
```

## Gotchas

- **At-least-once, not exactly-once.** Design consumers to be idempotent
  (e.g. `UPSERT` on `orderId`, or dedupe on `messageId`) — a redelivered
  message is normal operation, not a bug.
- **Ack deadline vs. processing time.** If your consumer takes longer than
  `--ack-deadline` to process a message, Pub/Sub redelivers it *while the
  first copy is still being processed* — extend the deadline
  (`modifyAckDeadline` in client libraries) for slow handlers instead of just
  raising the static default.
- **Unacked messages accumulate and cost money.** A subscription with no
  running consumer silently piles up backlog — the Cloud Monitoring metric
  `subscription/num_undelivered_messages` is how you catch a dead consumer
  before the backlog becomes a bill or a retention-expiry data-loss event.
- **Message retention has a default and a cap.** Undelivered messages are
  kept for 7 days by default (configurable up to 31 with
  `--message-retention-duration`); past that, they're gone even if never
  acknowledged.
- **Push endpoints must ack via HTTP status**, not response body — any
  non-2xx (including a 500 from an unhandled exception) is treated as a
  failure and triggers redelivery/retry backoff.

## Cleanup

```bash
gcloud pubsub subscriptions delete orders-email-sub
gcloud pubsub subscriptions delete orders-webhook-sub
gcloud pubsub subscriptions delete orders-ordered-sub
gcloud pubsub topics delete orders
gcloud pubsub topics delete orders-dlq
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud pubsub topics create` | Create a message channel. |
| `gcloud pubsub subscriptions create --topic=` | Attach a pull subscription to a topic. |
| `gcloud pubsub subscriptions create --push-endpoint=` | Attach a push (HTTP) subscription. |
| `gcloud pubsub topics publish --message=` | Publish a message. |
| `gcloud pubsub subscriptions pull --auto-ack` | Pull and acknowledge messages manually. |
| `gcloud pubsub subscriptions update --dead-letter-topic=` | Route repeatedly-failed messages to a DLQ. |
| `gcloud pubsub subscriptions create --enable-message-ordering` | Guarantee per-ordering-key delivery order. |

## Exercise

Create a topic with two subscriptions: one plain pull subscription and one
with a dead-letter topic and `--max-delivery-attempts=5`. Publish a few
messages, pull-without-acking from the second subscription repeatedly, and
confirm the message eventually lands in the dead-letter topic's own
subscription. Then create an ordering-key subscription, publish three
messages with the same `--ordering-key`, and confirm they pull back in the
order you sent them.
