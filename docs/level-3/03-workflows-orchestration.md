# 03 · Cloud Workflows & Orchestration

Pub/Sub (Level 2) decouples services with events, but some processes need
explicit, ordered steps with branching, retries, and error handling —
"call API A, then B with A's result, and if B fails, call C instead."
**Cloud Workflows** is GCP's serverless orchestrator for exactly that: a
YAML/JSON-defined sequence of steps that can call HTTP endpoints, GCP APIs,
and other workflows, with built-in retry and error-handling constructs.

## Defining a workflow

```yaml
# order-workflow.yaml
main:
  params: [input]
  steps:
    - init:
        assign:
          - order_id: ${input.order_id}
    - reserve_inventory:
        call: http.post
        args:
          url: https://inventory-svc-abc.a.run.app/reserve
          body:
            order_id: ${order_id}
        result: reserve_result
    - check_reservation:
        switch:
          - condition: ${reserve_result.body.status == "ok"}
            next: charge_payment
        next: release_and_fail
    - charge_payment:
        call: http.post
        args:
          url: https://payments-svc-abc.a.run.app/charge
          body:
            order_id: ${order_id}
        result: payment_result
        next: return_success
    - release_and_fail:
        call: http.post
        args:
          url: https://inventory-svc-abc.a.run.app/release
          body:
            order_id: ${order_id}
        next: fail
    - return_success:
        return: ${payment_result.body}
    - fail:
        raise: "inventory reservation failed"
```

```bash
gcloud workflows deploy order-workflow \
  --source=order-workflow.yaml \
  --location=us-central1

gcloud workflows execute order-workflow \
  --location=us-central1 \
  --data='{"order_id": "ORD-1001"}'
```

```bash
gcloud workflows executions describe EXECUTION_ID \
  --workflow=order-workflow --location=us-central1
# state: SUCCEEDED
# result: '{"charge_id":"ch_9f3...","status":"charged"}'
```

## Retries and error handling

```yaml
    - reserve_inventory:
        try:
          call: http.post
          args:
            url: https://inventory-svc-abc.a.run.app/reserve
            body:
              order_id: ${order_id}
          result: reserve_result
        retry:
          predicate: ${http.default_retry_predicate}
          max_retries: 3
          backoff:
            initial_delay: 1
            max_delay: 10
            multiplier: 2
        except:
          as: e
          steps:
            - log_failure:
                call: sys.log
                args:
                  text: ${"reserve failed: " + e.message}
                  severity: ERROR
            - fail_step:
                next: release_and_fail
```

`http.default_retry_predicate` retries on `429`, `5xx`, and connection
errors — not on `4xx` client errors, which is usually what you want since
retrying a malformed request just wastes time and money identically each
attempt.

**Gotcha — steps execute with a per-workflow execution deadline.** By
default a workflow execution can run up to one year for standard workflows,
but individual `call: http.*` steps carry their own default timeout
(shorter). Long-polling patterns need an explicit `timeout` argument on the
call, or the step fails well before the outer workflow would.

**Gotcha — `switch` needs an explicit fallthrough.** If no `condition`
matches and no `next` is set outside the switch block, the workflow throws
a runtime error rather than silently continuing — always give `switch` a
default `next` (as `check_reservation` does above with `release_and_fail`)
or an explicit unmatched-case branch.

## Workflows vs. Cloud Composer vs. Eventarc

| | Cloud Workflows | Cloud Composer (Airflow) | Eventarc |
|---|---|---|---|
| Model | Explicit step sequence, sync/async HTTP calls | DAG of scheduled/dependent tasks | Reactive: event in → trigger fires |
| Best for | Request/response orchestration across services | Data pipeline scheduling, backfills | Wiring GCP service events to a handler |
| Infra to manage | None (serverless) | A managed Airflow environment (persistent, billed) | None (serverless) |
| Typical trigger | API call, Pub/Sub message, Scheduler | Cron / sensor-based | GCS object created, Pub/Sub, Cloud Audit Log events |

Workflows is the right tool when you need explicit control flow and can
tolerate calling out to HTTP endpoints; Composer is right for data-pipeline
DAGs with many interdependent scheduled tasks; Eventarc (Level 4) is right
for "when X happens in GCP, run Y" without hand-rolled polling.

## IAM for calling other services

The workflow's own service account needs permission to invoke whatever it
calls — Cloud Run services, other workflows, etc.

```bash
gcloud run services add-iam-policy-binding inventory-svc \
  --region=us-central1 \
  --member="serviceAccount:workflow-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

gcloud workflows deploy order-workflow \
  --source=order-workflow.yaml \
  --location=us-central1 \
  --service-account=workflow-sa@my-project.iam.gserviceaccount.com
```

**Gotcha — default compute service account lacks `run.invoker` by
default.** If you don't specify `--service-account` at deploy time,
Workflows uses the project's default compute SA, which typically cannot
invoke a private Cloud Run service — calls fail with `403` even though the
workflow itself deployed fine. Always create and scope a dedicated
workflow SA.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud workflows deploy` | Create/update a workflow from a YAML/JSON source file. |
| `gcloud workflows execute` | Run a workflow with input data. |
| `gcloud workflows executions describe` | Check status/result of a specific execution. |
| `try` / `retry` / `except` | Built-in per-step retry and error-handling. |
| `http.default_retry_predicate` | Standard retry-on-5xx/429 policy. |
| `roles/run.invoker` on the workflow's SA | Required to call a private Cloud Run service. |

## How It Actually Works

Cloud Workflows executes a YAML/JSON-defined state machine where each
step is checkpointed to durable storage before the next one runs — this
is what makes workflow execution resilient to the underlying compute
being recycled mid-run: if the execution engine's worker restarts, it
resumes from the last checkpointed step rather than replaying the whole
workflow from scratch, and (unlike a plain script) a long-running `wait`
or external callback step can pause execution for hours or days without
holding any compute resource at all — the workflow's state simply sits
serialized in storage until the triggering event or timer fires. This
checkpoint-based execution is also why workflow steps calling external
APIs need explicit retry policies configured rather than relying on
try/catch semantics from a general-purpose language: a step that fails
after the call already had a side effect (e.g., a message already
published) will re-execute that step on retry unless you've made the
downstream call idempotent, the same at-least-once problem that shows up
in Pub/Sub.

## Exercise

Write a workflow with three HTTP steps calling three different Cloud Run
services (they can be stub services that just echo their input), where step
2 uses `try`/`retry`/`except` and step 3 only runs on success via `switch`.
Deploy it with a dedicated service account holding `roles/run.invoker` on
all three services, execute it, and inspect `gcloud workflows executions
describe` to confirm the retry step's behavior when you deliberately break
one service's URL.
