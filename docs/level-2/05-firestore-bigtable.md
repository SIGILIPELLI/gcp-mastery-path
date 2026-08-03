# 05 · Firestore & Bigtable Deep Dive

Level 1's Cloud SQL module covered relational storage. Not everything fits a
table with a fixed schema, and not everything needs SQL joins — this module
covers two of GCP's NoSQL databases, built for very different shapes of
problem: **Firestore** (serverless documents, built for app backends) and
**Bigtable** (wide-column, built for massive, low-latency throughput).

## Choosing between them

| | Firestore | Bigtable |
|---|---|---|
| Data model | Documents (JSON-like) in collections | Wide-column, rows keyed by a single row key |
| Scale target | App-sized: thousands to millions of docs | Petabyte-scale, millions of ops/sec |
| Query model | Rich queries, indexes, real-time listeners | Row-key range scans only — no secondary-index queries |
| Ops model | Fully serverless, zero capacity planning | You choose node count/type; scales but isn't free-tier friendly |
| Typical use | Mobile/web app backend, user profiles, carts | Time-series/IoT telemetry, ad-tech, analytics ingestion |
| Pricing | Per read/write/delete + storage | Per node-hour + storage, regardless of traffic |

The short version: if you're building an app and aren't sure, start with
Firestore. Reach for Bigtable only once you have a specific, very-high-volume
workload (millions of writes/sec, single-row-key access patterns) that
Firestore's per-operation pricing and query model don't fit.

## Firestore: enable and create data

Firestore has two modes chosen once, at project creation, and not changeable
afterward: **Native mode** (the modern document/real-time model) and
**Datastore mode** (legacy compatibility). New projects should always use
Native mode.

```bash
gcloud services enable firestore.googleapis.com

gcloud firestore databases create \
  --location=nam5 \
  --type=firestore-native
```

```bash
gcloud firestore databases list
# NAME                                        LOCATION  TYPE
# projects/gcp-mastery-path-123/databases/(default)  nam5  FIRESTORE_NATIVE
```

## Firestore: documents and collections

Firestore has no `gcloud` command for writing individual documents — reads
and writes go through a client library (or the console/emulator) since
documents are arbitrary JSON-like structures, not something a flag-based CLI
models well. The structure looks like:

```
users (collection)
 └── uid_abc123 (document)
      ├── name: "Priya"
      ├── plan: "pro"
      └── orders (subcollection)
           └── order_001 (document)
                ├── total: 42.50
                └── items: [...]
```

```bash
# Export existing data for backup/inspection (works via gcloud)
gcloud firestore export gs://gcp-mastery-path-123-firestore-backups/2026-08-03 \
  --collection-ids=users
```

```text
Waiting for [projects/gcp-mastery-path-123/databases/(default)/operations/...] to finish...done.
metadata:
  '@type': type.googleapis.com/google.firestore.admin.v1.ExportDocumentsMetadata
  operationState: SUCCESSFUL
```

## Firestore: indexes and security rules

Every query that filters/sorts on more than one field needs a **composite
index**, defined declaratively and deployed:

```yaml
# firestore.indexes.json
{
  "indexes": [
    {
      "collectionGroup": "orders",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    }
  ]
}
```

```bash
gcloud firestore indexes composite create \
  --collection-group=orders \
  --field-config=field-path=status,order=ascending \
  --field-config=field-path=createdAt,order=descending
```

Security rules — not IAM — are how Firestore controls access from client
apps (mobile/web SDKs bypass IAM entirely and talk to Firestore directly):

```
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

```bash
gcloud firestore databases update --database='(default)' --backup-schedule="0 3 * * *"
```

## Bigtable: instance and table

Bigtable is provisioned, not serverless — you pick node count and cluster
placement up front:

```bash
gcloud services enable bigtable.googleapis.com bigtableadmin.googleapis.com

gcloud bigtable instances create telemetry-instance \
  --display-name="Telemetry Instance" \
  --cluster-config=id=telemetry-cluster,zone=us-central1-b,nodes=1
```

```bash
gcloud bigtable instances list
# INSTANCE_NAME         DISPLAY_NAME          STATE
# telemetry-instance    Telemetry Instance    READY
```

```bash
# cbt is the Bigtable-specific CLI (installed via gcloud components)
gcloud components install cbt

cbt -instance=telemetry-instance createtable sensor-readings
cbt -instance=telemetry-instance createfamily sensor-readings metrics
```

## Bigtable: row keys are the whole design

Bigtable has exactly one index: the row key, sorted lexicographically. There
are no secondary indexes and no ad-hoc `WHERE` clauses — query performance is
entirely a function of row-key design.

```bash
# Row key: reverse-timestamp prefix keeps recent data together for range scans
cbt -instance=telemetry-instance set sensor-readings \
  sensor42#2026-08-03T10:00:00Z metrics:temp=21.4

cbt -instance=telemetry-instance read sensor-readings \
  prefix=sensor42#
```

```text
----------------------------------------
sensor42#2026-08-03T10:00:00Z
  metrics:temp                    @ 2026/08/03-10:00:00.000000
    "21.4"
```

A row key like a raw incrementing integer or a plain timestamp creates
**hotspotting**: all new writes land on the same node/tablet, capping your
real throughput far below what more nodes should provide. Prefixing with a
well-distributed field (device ID, hash) before the timestamp is the standard
fix.

## Gotchas

- **Firestore mode is permanent.** Native vs. Datastore mode is chosen once
  when the database is created and cannot be switched later — get it right
  the first time (Native, for anything new).
- **Firestore composite indexes must be pre-declared**, not created
  automatically at query time — a query that needs one you haven't deployed
  fails with an error that includes a direct console link to auto-create it,
  which is worth clicking during development but should be checked into
  `firestore.indexes.json` for real deployments.
- **Firestore security rules ≠ IAM.** IAM controls access from your backend
  (using service-account credentials); security rules control access from
  client SDKs that connect directly with a user's Firebase Auth token — both
  layers exist independently and neither substitutes for the other.
- **Bigtable bills per node-hour, not per request.** An idle 3-node cluster
  costs the same as a fully loaded one — this is the single biggest reason
  Bigtable is the wrong default for anything but sustained high-throughput
  workloads.
- **Bigtable row-key design is the entire performance model.** There's no
  query planner to bail you out of a bad key — hotspotting from sequential
  keys is the most common real-world Bigtable production incident.

## Cleanup

```bash
gcloud firestore indexes composite list --format="value(name)" | \
  xargs -I{} gcloud firestore indexes composite delete {} --quiet

cbt -instance=telemetry-instance deletetable sensor-readings
gcloud bigtable instances delete telemetry-instance --quiet
```

Firestore's default database itself cannot be deleted through most projects'
lifecycle — for a learning project, deleting the individual collections/
documents (or the whole GCP project) is the practical cleanup path.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud firestore databases create --type=firestore-native` | Provision the (one-time, permanent) Firestore mode. |
| `gcloud firestore export gs://<bucket>/<path>` | Back up Firestore data to Cloud Storage. |
| `gcloud firestore indexes composite create` | Declare a multi-field composite index. |
| `gcloud bigtable instances create --cluster-config=` | Provision a Bigtable instance and its first cluster. |
| `cbt createtable` / `createfamily` | Create a table and column family. |
| `cbt set` / `read` | Write/read rows via the Bigtable-specific CLI. |

## Exercise

Create a Firestore Native database, then (using the Firestore emulator or
console, since `gcloud` doesn't write individual documents) add a handful of
documents to a `users` collection with an `orders` subcollection. Separately,
create a single-node Bigtable instance, create a table with one column
family, and write 3-4 rows using row keys with a device-ID prefix before a
timestamp. Use `cbt read ... prefix=` to confirm the range scan returns only
that device's rows in time order.
