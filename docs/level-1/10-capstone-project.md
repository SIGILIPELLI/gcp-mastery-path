# 10 · Capstone Project

Time to combine everything from this level into one small but real
end-to-end application: a **static frontend on Cloud Storage**, a **Cloud
Run API backend**, and a **Cloud SQL database**, wired together with the
IAM, networking, and monitoring practices from earlier modules.

!!! danger "Read the teardown section before you start"
    This project uses billable resources (a Cloud SQL instance in
    particular). Jump to [Teardown](#teardown) now and skim it, so you know
    exactly what you'll need to delete before you begin — don't let this be
    the one project you forget to clean up.

## What you're building

```
Browser
  │
  ▼
Cloud Storage (static site: index.html + JS)
  │  fetch("https://<cloud-run-url>/api/notes")
  ▼
Cloud Run (Flask API: "notes" app)
  │  Cloud SQL Auth Proxy / Unix socket
  ▼
Cloud SQL (PostgreSQL: notes table)
```

A minimal "notes" app: the frontend lists notes and lets you add one, the
backend is a small REST API, the database persists the notes.

## Step 1 — Database

```bash
gcloud sql instances create capstone-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-central1

gcloud sql databases create notesdb --instance=capstone-db

gcloud sql users create notesapp \
  --instance=capstone-db \
  --password=CHANGE-ME-DEMO-ONLY
```

```sql
-- Run once via `gcloud sql connect capstone-db --user=postgres`, then \c notesdb
CREATE TABLE notes (
    id SERIAL PRIMARY KEY,
    body TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

## Step 2 — Backend API (Cloud Run)

```python
# app.py
import os
from flask import Flask, jsonify, request
import psycopg2
from psycopg2.extras import RealDictCursor

app = Flask(__name__)

def get_conn():
    return psycopg2.connect(
        host=f"/cloudsql/{os.environ['INSTANCE_CONNECTION_NAME']}",
        dbname=os.environ["DB_NAME"],
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASSWORD"],
    )

@app.route("/api/notes", methods=["GET"])
def list_notes():
    with get_conn() as conn, conn.cursor(cursor_factory=RealDictCursor) as cur:
        cur.execute("SELECT id, body, created_at FROM notes ORDER BY id DESC")
        return jsonify(cur.fetchall())

@app.route("/api/notes", methods=["POST"])
def add_note():
    body = request.get_json(force=True).get("body", "").strip()
    if not body:
        return jsonify({"error": "body is required"}), 400
    with get_conn() as conn, conn.cursor(cursor_factory=RealDictCursor) as cur:
        cur.execute(
            "INSERT INTO notes (body) VALUES (%s) RETURNING id, body, created_at",
            (body,),
        )
        conn.commit()
        return jsonify(cur.fetchone()), 201

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

```text
# requirements.txt
flask==3.*
psycopg2-binary==2.*
```

```dockerfile
# Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```

Deploy, wiring in the Cloud SQL connection and (for this demo) plain
environment variables — in anything beyond a learning capstone, pull the
password from Secret Manager instead:

```bash
gcloud run deploy capstone-api \
  --source=. \
  --region=us-central1 \
  --allow-unauthenticated \
  --add-cloudsql-instances=PROJECT_ID:us-central1:capstone-db \
  --set-env-vars=INSTANCE_CONNECTION_NAME=PROJECT_ID:us-central1:capstone-db,DB_NAME=notesdb,DB_USER=notesapp,DB_PASSWORD=CHANGE-ME-DEMO-ONLY
```

```bash
API_URL=$(gcloud run services describe capstone-api --region=us-central1 --format='value(status.url)')
curl -X POST "$API_URL/api/notes" -H "Content-Type: application/json" -d '{"body":"first note"}'
curl "$API_URL/api/notes"
```

## Step 3 — Frontend (Cloud Storage static site)

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>Capstone Notes</title></head>
<body>
  <h1>Notes</h1>
  <form id="form"><input id="body" placeholder="New note"><button>Add</button></form>
  <ul id="list"></ul>
  <script>
    const API_URL = "REPLACE_WITH_YOUR_CLOUD_RUN_URL";

    async function load() {
      const res = await fetch(`${API_URL}/api/notes`);
      const notes = await res.json();
      document.getElementById("list").innerHTML =
        notes.map(n => `<li>${n.body}</li>`).join("");
    }

    document.getElementById("form").addEventListener("submit", async (e) => {
      e.preventDefault();
      const body = document.getElementById("body").value;
      await fetch(`${API_URL}/api/notes`, {
        method: "POST",
        headers: {"Content-Type": "application/json"},
        body: JSON.stringify({body}),
      });
      document.getElementById("body").value = "";
      load();
    });

    load();
  </script>
</body>
</html>
```

Replace `REPLACE_WITH_YOUR_CLOUD_RUN_URL` with the `$API_URL` value from
Step 2, then publish it:

```bash
gcloud storage buckets create gs://capstone-notes-frontend-PROJECT_ID \
  --location=us-central1 \
  --uniform-bucket-level-access

gcloud storage cp index.html gs://capstone-notes-frontend-PROJECT_ID/

gcloud storage buckets add-iam-policy-binding gs://capstone-notes-frontend-PROJECT_ID \
  --member=allUsers \
  --role=roles/storage.objectViewer

gcloud storage buckets update gs://capstone-notes-frontend-PROJECT_ID \
  --web-main-page-suffix=index.html
```

The API needs to accept requests from the bucket's origin — for a learning
capstone, add a permissive CORS header in the Flask app
(`response.headers["Access-Control-Allow-Origin"] = "*"` on every response);
in production you'd restrict this to the exact frontend origin.

Visit `https://storage.googleapis.com/capstone-notes-frontend-PROJECT_ID/index.html`,
add a note, and confirm it round-trips through Cloud Run into Cloud SQL and
back.

## Step 4 — Observe it

Reuse Module 8's habits on your own project:

```bash
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="capstone-api"' \
  --limit=10

gcloud monitoring time-series list \
  --filter='metric.type="run.googleapis.com/request_count" AND resource.labels.service_name="capstone-api"' \
  --interval-start-time="$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --interval-end-time="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

## Teardown

**This is not optional.** Cloud SQL instances in particular bill continuously
while they exist, free trial credit or not — delete every piece, in this
order:

```bash
# 1. Cloud Run service
gcloud run services delete capstone-api --region=us-central1 --quiet

# 2. Frontend bucket and its contents
gcloud storage rm --recursive gs://capstone-notes-frontend-PROJECT_ID
gcloud storage buckets delete gs://capstone-notes-frontend-PROJECT_ID --quiet

# 3. Cloud SQL instance -- the highest-cost resource in this project
gcloud sql instances delete capstone-db --quiet

# 4. Confirm nothing capstone-related is still running
gcloud run services list
gcloud sql instances list
gcloud storage buckets list --filter="name:capstone"
gcloud compute instances list
```

The last four commands should all come back **empty** (or show only
resources from earlier modules that you've separately confirmed you still
want). If `gcloud sql instances list` shows anything, billing for it
continues — don't stop at step 3 without verifying with step 4.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud sql instances create/delete` | Provision/remove the database — highest ongoing cost, delete first priority. |
| `gcloud run deploy --source=. --add-cloudsql-instances=` | Deploy the API wired to Cloud SQL. |
| `gcloud storage buckets create --uniform-bucket-level-access` | Create the frontend bucket. |
| `gcloud storage buckets update --web-main-page-suffix=` | Enable static website serving. |
| `gcloud run services delete` | Tear down the backend. |
| `gcloud storage buckets delete` | Tear down the frontend bucket (must be empty first). |
| `gcloud sql instances list` / `gcloud run services list` | Verify teardown left nothing running. |

## Exercise

Extend the notes app with a `DELETE /api/notes/<id>` endpoint and a delete
button per note in the frontend. Redeploy both pieces, verify delete works
end-to-end from the browser, and then — before you consider this capstone
done — run every command in the Teardown section and confirm all four
verification commands come back empty.
