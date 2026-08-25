# 03 · Data Analytics (BigQuery, Dataflow, Dataproc)

GCP's analytics stack has three layers most orgs end up using together:
**BigQuery** for warehousing and SQL analytics, **Dataflow** for
unified batch/streaming pipelines (Apache Beam), and **Dataproc** for
managed Spark/Hadoop when you have existing Spark jobs to lift-and-shift.

## BigQuery: partitioning and clustering

Querying an unpartitioned multi-terabyte table scans the whole thing every
time — expensive and slow. Partitioning by date and clustering by a
high-cardinality filter column fixes both.

```sql
CREATE TABLE orders.events
(
  event_id STRING,
  order_id STRING,
  customer_id STRING,
  event_type STRING,
  event_ts TIMESTAMP
)
PARTITION BY DATE(event_ts)
CLUSTER BY customer_id;
```

```bash
bq query --use_legacy_sql=false --dry_run \
'SELECT * FROM orders.events WHERE DATE(event_ts) = "2026-08-20" AND customer_id = "C-4471"'
# Query successfully validated. Assuming default project my-project.
# Bytes processed: 42000000 (partition pruning applied)
```

Compare that `--dry_run` byte estimate against the same query without the
`WHERE DATE(...)` predicate — the difference shows exactly what
partitioning saved, since BigQuery bills by bytes scanned.

**Gotcha — partition pruning requires filtering on the partition column
directly, not a derived expression on a different column.** `WHERE
event_ts >= TIMESTAMP("2026-08-20")` prunes correctly; `WHERE
DATE(some_other_timestamp_col) = "2026-08-20"` does not, because it's not
filtering the actual partitioning column. Always dry-run a query after
changing filter logic to confirm pruning is still happening.

**Gotcha — a table can have at most 4,000 partitions by default** with
daily partitioning; for very long-lived tables, consider monthly
partitioning or partition expiration:

```bash
bq update --time_partitioning_expiration 7776000 orders.events   # 90 days
```

## Dataflow: a streaming pipeline

Dataflow runs Apache Beam pipelines with autoscaling workers, handling both
bounded (batch) and unbounded (streaming) data with the same programming
model.

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

options = PipelineOptions(
    runner="DataflowRunner",
    project="my-project",
    region="us-central1",
    temp_location="gs://my-project-dataflow-temp/tmp",
    streaming=True,
)

with beam.Pipeline(options=options) as p:
    (
        p
        | "ReadPubSub" >> beam.io.ReadFromPubSub(topic="projects/my-project/topics/orders-events")
        | "ParseJSON" >> beam.Map(parse_event)
        | "WindowInto" >> beam.WindowInto(beam.window.FixedWindows(60))
        | "CountPerWindow" >> beam.combiners.Count.PerElement()
        | "WriteBQ" >> beam.io.WriteToBigQuery(
              "my-project:orders.event_counts",
              write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
          )
    )
```

```bash
python pipeline.py
gcloud dataflow jobs list --region=us-central1 --status=active
# JOB_ID       NAME              TYPE       STATE
# 2026-08...   orders-counter    Streaming  Running
```

```bash
gcloud dataflow jobs show JOB_ID --region=us-central1 --format="value(currentState)"
# JOB_STATE_RUNNING
```

**Gotcha — streaming jobs run (and bill) continuously until explicitly
cancelled**, unlike batch jobs which finish on their own:

```bash
gcloud dataflow jobs cancel JOB_ID --region=us-central1
```

A forgotten streaming pipeline from a demo is a recurring line item, not a
one-time charge — always list active jobs (`--status=active`) as part of
routine cost review.

**Gotcha — windowing without a trigger strategy can delay results
indefinitely for late data.** `FixedWindows(60)` alone waits for the
watermark to pass the window's end before firing; a `beam.trigger.AfterProcessingTime`
or allowed-lateness setting is usually needed for real streaming latency
requirements, or results lag behind wall-clock time more than expected.

## Dataproc: managed Spark

For existing Spark/Hadoop jobs, Dataproc gives a managed, ephemeral cluster
model — spin up, run the job, tear down, pay only for the run.

```bash
gcloud dataproc clusters create analytics-cluster \
  --region=us-central1 \
  --num-workers=4 \
  --worker-machine-type=n2-standard-4 \
  --image-version=2.1-debian11

gcloud dataproc jobs submit pyspark gs://my-project/jobs/aggregate_orders.py \
  --cluster=analytics-cluster \
  --region=us-central1
```

```bash
gcloud dataproc clusters delete analytics-cluster --region=us-central1 -q
```

**Gotcha — ephemeral-per-job is the recommended pattern, not a persistent
cluster.** A cluster left running between jobs bills for idle worker time;
use `--max-idle=30m` (auto-delete after idle) or wrap create → submit →
delete in a Workflow/Cloud Build pipeline so nothing lingers:

```bash
gcloud dataproc clusters create analytics-cluster \
  --region=us-central1 --num-workers=4 --max-idle=30m
```

## Choosing between them

| | BigQuery | Dataflow | Dataproc |
|---|---|---|---|
| Model | SQL over managed storage | Beam pipeline (batch or streaming) | Managed Spark/Hadoop |
| Best for | Ad hoc analytics, dashboards, warehousing | ETL/streaming transforms, especially GCP-native sources | Lift-and-shift existing Spark code |
| Ops overhead | None (fully serverless) | None (autoscaled workers) | Cluster lifecycle to manage (even if ephemeral) |

## Cheat sheet

| Command | Purpose |
|---|---|
| `PARTITION BY DATE(...)` / `CLUSTER BY` | Reduce bytes scanned, and cost, on large tables. |
| `bq query --dry_run` | Estimate bytes scanned before running a query for real. |
| `gcloud dataflow jobs list --status=active` | Catch forgotten, still-billing streaming jobs. |
| `gcloud dataflow jobs cancel` | Stop a streaming pipeline explicitly. |
| `gcloud dataproc clusters create --max-idle=` | Auto-delete an idle ephemeral Spark cluster. |
| `gcloud dataproc jobs submit pyspark` | Run a Spark job against a Dataproc cluster. |

## Exercise

Design a BigQuery table schema for an events table partitioned by day and
clustered by `customer_id`. Write two queries — one that prunes partitions
correctly and one that accidentally doesn't (same intent, different filter
expression) — and describe how `bq query --dry_run` would show the
difference in bytes scanned. Then sketch a Dataflow pipeline reading from
Pub/Sub and writing 60-second windowed aggregates to that table, noting
what you'd add to handle late-arriving events.
