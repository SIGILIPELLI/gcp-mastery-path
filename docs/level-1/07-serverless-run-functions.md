# 07 · Cloud Functions & Cloud Run

GCP has two serverless compute products that solve overlapping but distinct
problems. **Cloud Functions** runs a single function in response to an
event. **Cloud Run** runs a container (any language, any framework) and
scales it, including to zero, in response to HTTP requests or events. This
module deploys one of each.

## Cloud Functions: a single event-triggered function

```bash
mkdir hello-function && cd hello-function
```

```python
# main.py
import functions_framework

@functions_framework.http
def hello(request):
    name = request.args.get("name", "World")
    return f"Hello, {name}, from Cloud Functions!\n"
```

```text
# requirements.txt
functions-framework==3.*
```

Deploy it (2nd gen Cloud Functions run on top of Cloud Run under the hood):

```bash
gcloud functions deploy hello-function \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=hello \
  --trigger-http \
  --allow-unauthenticated
```

```bash
gcloud functions call hello-function --region=us-central1 --data='{}'
# or, since it's HTTP-triggered:
curl "$(gcloud functions describe hello-function --region=us-central1 --format='value(serviceConfig.uri)')?name=GCP"
# Hello, GCP, from Cloud Functions!
```

### Event triggers besides HTTP

Cloud Functions also fires on non-HTTP events — a new object landing in a
Cloud Storage bucket, a Pub/Sub message, a Firestore write:

```bash
gcloud functions deploy on-new-upload \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=process_upload \
  --trigger-bucket=gcp-mastery-path-123-assets
```

## Cloud Run: any container, scale-to-zero

Cloud Run takes a container image and runs it as a fully-managed, scale-
to-zero HTTP (or worker) service. It's the better fit once your app is more
than one function — a whole web framework, multiple routes, background
threads, or a language/runtime Cloud Functions doesn't support directly.

```python
# app.py
import os
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    name = os.environ.get("NAME", "World")
    return f"Hello, {name}, from Cloud Run!\n"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
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

```text
# requirements.txt
flask==3.*
```

Deploy directly from source — Cloud Build compiles the container for you, no
local Docker required:

```bash
gcloud run deploy hello-run \
  --source=. \
  --region=us-central1 \
  --allow-unauthenticated \
  --set-env-vars=NAME=GCP
```

```bash
curl "$(gcloud run services describe hello-run --region=us-central1 --format='value(status.url)')"
# Hello, GCP, from Cloud Run!
```

If you already have an image built and pushed (e.g. to Artifact Registry),
deploy that instead of `--source`:

```bash
gcloud run deploy hello-run \
  --image=us-central1-docker.pkg.dev/PROJECT_ID/repo/hello-run:latest \
  --region=us-central1 \
  --allow-unauthenticated
```

## Scaling, concurrency, and cost

Both scale automatically with demand and, unlike a Compute Engine VM, bill
only for actual request-processing time (Cloud Run also supports an
always-on minimum-instances setting when cold starts matter more than cost).

```bash
gcloud run services update hello-run \
  --region=us-central1 \
  --min-instances=0 \
  --max-instances=5 \
  --concurrency=80
```

- `--min-instances=0` allows scale-to-zero (no traffic → no running
  instances → no charge) — the default and the free-tier-friendly choice.
- `--concurrency` controls how many simultaneous requests one instance
  handles before Cloud Run starts another.

## Choosing between them

| | Cloud Functions | Cloud Run |
|---|---|---|
| Unit of deployment | One function | A container (any language/framework) |
| Best for | Small, single-purpose event handlers | Full apps, APIs, multiple routes, custom runtimes |
| Cold start | Small | Small–moderate (depends on image) |
| Non-HTTP triggers | Built-in (Storage, Pub/Sub, Firestore, etc.) | Via Eventarc, or HTTP push from Pub/Sub |
| Local dev | Functions Framework | Any container tooling you already use |

## Cleanup

```bash
gcloud functions delete hello-function --region=us-central1 --quiet
gcloud functions delete on-new-upload --region=us-central1 --quiet
gcloud run services delete hello-run --region=us-central1 --quiet
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud functions deploy --gen2 --trigger-http` | Deploy an HTTP-triggered function. |
| `gcloud functions deploy --trigger-bucket=` | Deploy a function triggered by Cloud Storage events. |
| `gcloud functions call` | Invoke a function directly (for non-HTTP or quick tests). |
| `gcloud run deploy --source=.` | Build (via Cloud Build) and deploy a container from source. |
| `gcloud run deploy --image=` | Deploy a pre-built container image. |
| `gcloud run services update --min-instances/--max-instances` | Tune scaling behavior. |
| `gcloud run services describe --format='value(status.url)'` | Get a service's public URL. |
| `gcloud functions delete` / `gcloud run services delete` | Tear down. |

## Exercise

Deploy an HTTP Cloud Function that returns the current server time as JSON.
Separately, containerize a tiny Flask (or Express, or any framework you
prefer) app with one extra route and deploy it to Cloud Run with
`--min-instances=0`. Curl both, confirm they respond, then delete both.
