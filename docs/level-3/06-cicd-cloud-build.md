# 06 · CI/CD (Cloud Build, Artifact Registry)

**Cloud Build** runs a sequence of container-based build steps triggered by
a git push, and **Artifact Registry** stores the resulting images (and
other package types). Together they form GCP's native CI/CD path from
commit to deployed revision.

## A build config

```yaml
# cloudbuild.yaml
steps:
  - name: gcr.io/cloud-builders/docker
    args:
      - build
      - -t
      - us-central1-docker.pkg.dev/$PROJECT_ID/app/orders:$SHORT_SHA
      - .
  - name: gcr.io/cloud-builders/docker
    args:
      - push
      - us-central1-docker.pkg.dev/$PROJECT_ID/app/orders:$SHORT_SHA
  - name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: gcloud
    args:
      - run
      - deploy
      - orders-api
      - --image=us-central1-docker.pkg.dev/$PROJECT_ID/app/orders:$SHORT_SHA
      - --region=us-central1
images:
  - us-central1-docker.pkg.dev/$PROJECT_ID/app/orders:$SHORT_SHA
options:
  logging: CLOUD_LOGGING_ONLY
```

```bash
gcloud artifacts repositories create app \
  --repository-format=docker \
  --location=us-central1

gcloud builds submit --config=cloudbuild.yaml .
```

```bash
# Build log excerpt
# Step #0 - "docker": Successfully tagged us-central1-docker.pkg.dev/.../orders:a1b2c3d
# Step #1 - "docker": a1b2c3d: digest: sha256:... size: 1987
# Step #2 - "gcloud": Service [orders-api] revision [orders-api-00007-xyz] has been deployed
# DONE
```

`$SHORT_SHA` and `$PROJECT_ID` are Cloud Build **substitutions**, populated
automatically from the triggering commit — tagging images by commit SHA
(not `latest`) means every deployed revision maps back to an exact commit.

## Triggers on push

```bash
gcloud builds triggers create github \
  --repo-name=orders-service \
  --repo-owner=my-org \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml \
  --name=orders-main-trigger
```

```bash
gcloud builds triggers list
# NAME                  CREATE_TIME  STATUS
# orders-main-trigger   2026-01-10   
```

A push to `main` now kicks off `cloudbuild.yaml` automatically. Use a
separate trigger with a different `--branch-pattern` (e.g. `^release-.*$`)
pointed at a `cloudbuild-prod.yaml` for a distinct prod pipeline, rather
than branching logic inside one config.

**Gotcha — the Cloud Build service account needs explicit deploy
permissions.** Cloud Build's default SA
(`PROJECT_NUM@cloudbuild.gserviceaccount.com`) can build and push images out
of the box, but `gcloud run deploy` from within a build step fails with a
permission error until you grant it `roles/run.admin` and
`roles/iam.serviceAccountUser` (to act as the runtime SA):

```bash
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:PROJECT_NUM@cloudbuild.gserviceaccount.com" \
  --role="roles/run.admin"

gcloud iam service-accounts add-iam-policy-binding \
  orders-runtime-sa@my-project.iam.gserviceaccount.com \
  --member="serviceAccount:PROJECT_NUM@cloudbuild.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"
```

## Artifact Registry vs. Container Registry

Container Registry (`gcr.io`) is the older, deprecated image store;
Artifact Registry (`*-docker.pkg.dev`) is its replacement and also supports
non-container formats — Maven, npm, Python, Go modules — in one place.

```bash
gcloud artifacts repositories create app-npm \
  --repository-format=npm \
  --location=us-central1

gcloud artifacts docker images list us-central1-docker.pkg.dev/my-project/app
# IMAGE                                              DIGEST         TAGS
# .../app/orders                                     sha256:a1b2..  a1b2c3d
```

**Gotcha — regional repos don't replicate automatically.** An Artifact
Registry repo is pinned to one region (or declared multi-region at
creation, not after). A Cloud Run service in `europe-west1` pulling from a
`us-central1` repo works but adds cross-region latency to every cold
start — create a repo per region you deploy into for latency-sensitive
services, and push the same tag to each.

## Build approvals and manual gates

For a prod pipeline, add an approval gate so a build doesn't deploy without
a human sign-off:

```bash
gcloud builds triggers create github \
  --repo-name=orders-service \
  --repo-owner=my-org \
  --branch-pattern="^release-.*$" \
  --build-config=cloudbuild-prod.yaml \
  --require-approval \
  --name=orders-prod-trigger
```

```bash
gcloud builds list --filter="status=PENDING_APPROVAL"
gcloud builds approve BUILD_ID
```

**Gotcha — approval only pauses execution, it doesn't pre-validate the
diff.** `--require-approval` stops the build before running any steps, but
whoever approves needs to have already reviewed the actual code change
(via the PR) — the approval gate itself shows no diff, just "run this
pipeline: yes/no."

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud builds submit --config=` | Run a build manually from local source. |
| `gcloud builds triggers create github` | Auto-run a build config on push/PR. |
| `$SHORT_SHA`, `$PROJECT_ID` | Built-in substitutions for commit-based tagging. |
| `roles/run.admin` + `roles/iam.serviceAccountUser` on Cloud Build SA | Required for a build step to deploy to Cloud Run. |
| `gcloud artifacts repositories create` | Create a regional image/package repo. |
| `--require-approval` / `gcloud builds approve` | Manual gate before a pipeline proceeds. |

## Exercise

Write a `cloudbuild.yaml` that builds an image, pushes it to a new Artifact
Registry Docker repo, and deploys it to Cloud Run — tagging by
`$SHORT_SHA`. Grant the Cloud Build SA the two roles it needs, submit the
build manually with `gcloud builds submit`, and confirm the deployed
revision's image digest matches what Artifact Registry shows via `gcloud
artifacts docker images list`.
