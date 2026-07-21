# 06 · Cloud SQL

**Cloud SQL** is GCP's fully-managed relational database service — it
handles patching, backups, replication, and failover for MySQL, PostgreSQL,
or SQL Server so you don't have to run and babysit the database server
yourself. This module creates a PostgreSQL instance, a database, a user, and
connects an app to it.

## Creating an instance

```bash
gcloud sql instances create gcp-mastery-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-central1 \
  --storage-size=10GB \
  --storage-type=HDD
```

- `--tier` picks the machine shape — `db-f1-micro` and `db-g1-small` are the
  smallest, cheapest shared-core tiers, good for learning and light dev use.
- Provisioning takes a few minutes; a Cloud SQL "instance" is really a small
  managed VM plus the database engine, backups, and networking around it.

```bash
gcloud sql instances describe gcp-mastery-db
gcloud sql instances list
```

## Databases and users

An instance can host multiple logical databases, each with its own users:

```bash
gcloud sql databases create appdb --instance=gcp-mastery-db

gcloud sql users create appuser \
  --instance=gcp-mastery-db \
  --password=CHANGE-ME-USE-SECRET-MANAGER
```

!!! tip "Don't hardcode passwords"
    The inline `--password` flag above is for a quick learning exercise
    only. In anything beyond that, generate/store the password in **Secret
    Manager** and have your app fetch it at runtime instead of embedding it
    in a command, script, or repo.

## Connecting: three ways

**1. Cloud SQL Auth Proxy** (recommended for local dev and for apps not
already on GCP's private network) — authenticates with IAM, so no need to
whitelist IPs or manage SSL certs yourself:

```bash
# Download once: https://cloud.google.com/sql/docs/postgres/sql-proxy
./cloud-sql-proxy PROJECT_ID:us-central1:gcp-mastery-db &

psql "host=127.0.0.1 port=5432 dbname=appdb user=appuser"
```

**2. Public IP + authorized networks** — simplest to reason about, weakest
default security; only ever open specific IPs, never `0.0.0.0/0`:

```bash
gcloud sql instances patch gcp-mastery-db \
  --authorized-networks=203.0.113.10/32

psql "host=$(gcloud sql instances describe gcp-mastery-db --format='value(ipAddresses[0].ipAddress)') \
  port=5432 dbname=appdb user=appuser"
```

**3. Private IP** — the instance gets an internal address inside your VPC
(via VPC peering) and is reachable only from resources on that network, with
no public exposure at all. This is the standard choice for production
workloads already running on Compute Engine, GKE, or Cloud Run in the same
VPC.

## Connecting from Cloud Run / Cloud Functions

Serverless compute connects via a **Unix socket** through the same Auth
Proxy mechanism, wired up with one flag at deploy time — no proxy binary or
IP allowlisting needed:

```bash
gcloud run deploy my-app \
  --image=gcr.io/PROJECT_ID/my-app \
  --add-cloudsql-instances=PROJECT_ID:us-central1:gcp-mastery-db \
  --set-env-vars=DB_SOCKET=/cloudsql/PROJECT_ID:us-central1:gcp-mastery-db
```

Your app's connection string then points at that socket path instead of a
host/port.

## Backups and point-in-time recovery

```bash
# Automated daily backups are on by default; trigger one on demand
gcloud sql backups create --instance=gcp-mastery-db

gcloud sql backups list --instance=gcp-mastery-db

# Restore a specific backup into a new instance (never overwrite the original blindly)
gcloud sql backups restore BACKUP_ID \
  --restore-instance=gcp-mastery-db-restored \
  --backup-instance=gcp-mastery-db
```

With point-in-time recovery enabled (adds write-ahead log retention), you
can restore to any moment within the retention window, not just a nightly
snapshot.

## Cleanup

```bash
gcloud sql instances delete gcp-mastery-db --quiet
```

Deleting the instance removes the database server, its databases, and its
automated backups — export anything you need to keep first with
`gcloud sql export sql`.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud sql instances create` | Create a managed database instance. |
| `gcloud sql instances list/describe` | View instance status and connection info. |
| `gcloud sql databases create` | Create a logical database on an instance. |
| `gcloud sql users create` | Create a database user. |
| `gcloud sql instances patch --authorized-networks=` | Allow a specific IP for public-IP connections. |
| `cloud-sql-proxy` | Local/app-side proxy authenticating via IAM, no IP allowlisting. |
| `--add-cloudsql-instances` (Cloud Run/Functions) | Wire serverless compute to a Cloud SQL instance via Unix socket. |
| `gcloud sql backups create/list/restore` | Manage on-demand and automated backups. |
| `gcloud sql instances delete` | Permanently delete an instance and its data. |

## Exercise

Create a `db-f1-micro` PostgreSQL instance, a database called `appdb`, and a
user. Connect to it using the Cloud SQL Auth Proxy and `psql`, create one
table and insert a row, then trigger an on-demand backup with
`gcloud sql backups create`. Confirm the backup appears in
`gcloud sql backups list`, then delete the instance.
