# 07 · Secret Manager & Config

Every module so far that needed a credential — a database password, an API
key — has quietly assumed it lives somewhere safe. **Secret Manager** is
that somewhere: a managed store for sensitive values, versioned, encrypted
at rest, and access-controlled through IAM rather than baked into
`app.yaml`, a Dockerfile, or (worse) committed to source control.

## Why not environment variables or a config file

Environment variables and plain config files are fine for non-sensitive
settings (a region name, a feature flag), but for actual secrets they have
real problems: they show up in `gcloud` describe output, crash logs, process
listings, and CI logs; they have no built-in versioning or rotation; and
revoking access means redeploying every consumer instead of changing an IAM
binding. Secret Manager fixes all four.

## Create a secret

```bash
gcloud services enable secretmanager.googleapis.com

echo -n "s3cr3t-db-password" | gcloud secrets create db-password \
  --data-file=- \
  --replication-policy="automatic"
```

```bash
gcloud secrets list
# NAME          CREATED              REPLICATION_POLICY  LOCATIONS
# db-password   2026-08-03T10:00:00  automatic            -
```

`--replication-policy="automatic"` lets Google choose replica regions; use
`--replication-policy="user-managed" --locations=us-central1,us-east1` if you
have data-residency requirements that restrict which regions may hold a copy.

## Versions

Every write to a secret creates a new, immutable **version** rather than
overwriting the value:

```bash
echo -n "n3w-r0tated-password" | gcloud secrets versions add db-password \
  --data-file=-
```

```bash
gcloud secrets versions list db-password
# NAME  STATE     CREATED
# 2     ENABLED   2026-08-03T11:00:00
# 1     ENABLED   2026-08-03T10:00:00
```

```bash
# Access the latest version (recommended for most cases)
gcloud secrets versions access latest --secret=db-password

# Access a specific version by number
gcloud secrets versions access 1 --secret=db-password
```

Old versions stay `ENABLED` (and readable) unless explicitly disabled or
destroyed — after rotating a credential, disable the old version so a stale
reference fails loudly instead of silently keeping working with a
credential you meant to retire:

```bash
gcloud secrets versions disable 1 --secret=db-password
```

## Granting access with IAM

Secret Manager access is per-secret, via IAM, not a shared master password:

```bash
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:web-app-sa@gcp-mastery-path-123.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

`roles/secretmanager.secretAccessor` grants read-only access to secret
*values* — it does not allow creating new secrets or versions
(`roles/secretmanager.admin` is the broader role for managing the secrets
themselves), which is the right split between an app's runtime identity and
whoever manages the secrets it consumes.

## Using a secret in Cloud Run

Rather than fetching a secret manually in application code, Cloud Run can
mount it directly as an environment variable at container start:

```bash
gcloud run deploy web-app \
  --image=us-docker.pkg.dev/gcp-mastery-path-123/repo/web-app:latest \
  --region=us-central1 \
  --set-secrets="DB_PASSWORD=db-password:latest" \
  --service-account=web-app-sa@gcp-mastery-path-123.iam.gserviceaccount.com
```

```bash
gcloud run services describe web-app --region=us-central1 \
  --format="value(spec.template.spec.containers[0].env)"
```

`db-password:latest` binds to whatever version is currently `latest` at
*container start time* — a running container does not automatically pick up
a newly added version; it needs a new revision (redeploy) to see it, which
is a deliberate design so secret rotation doesn't change behavior underneath
a running process mid-request.

## Using a secret in GKE

```bash
kubectl create secret generic db-password-k8s \
  --from-literal=password="$(gcloud secrets versions access latest --secret=db-password)"
```

For production GKE workloads, the **Secret Manager CSI driver** mounts
secrets directly from Secret Manager as files inside the pod — avoiding the
step above of copying a value into a native Kubernetes Secret, which is
itself only base64-encoded, not encrypted, by default:

```bash
gcloud container clusters update gcp-mastery-cluster \
  --region=us-central1 \
  --enable-secret-manager
```

## Terraform equivalent

```hcl
resource "google_secret_manager_secret" "db_password" {
  secret_id = "db-password"

  replication {
    auto {}
  }
}

resource "google_secret_manager_secret_version" "db_password_v1" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = var.db_password
}

resource "google_secret_manager_secret_iam_member" "web_app_access" {
  secret_id = google_secret_manager_secret.db_password.id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:web-app-sa@gcp-mastery-path-123.iam.gserviceaccount.com"
}
```

Passing `var.db_password` keeps the actual secret value out of the `.tf`
file itself — supply it via `terraform apply -var="db_password=..."` from a
secure input (CI secret store, not a committed `.tfvars` file).

## Gotchas

- **A committed `.tfvars` file defeats the point.** Feeding Terraform a
  secret value via a variable is only safer than hardcoding it if that
  variable's value never itself lands in version control.
- **`latest` is convenient but not repeatable.** For anything needing an
  audit trail of exactly which secret version a given deployment used, pin
  to an explicit version number rather than `latest`.
- **Disabling ≠ deleting.** `gcloud secrets versions disable` blocks access
  but keeps the value recoverable (`enable` reverses it); `destroy` is
  irreversible and should be reserved for confirmed-compromised values.
- **IAM binding is per-secret, not global.** A service account with access
  to `db-password` has no access to any other secret unless separately
  granted — this is a feature (least privilege) but means access sprawls
  across many `add-iam-policy-binding` calls in a project with many secrets.
- **Cloud Run secret env vars need a new revision to pick up rotation.**
  Rotating the secret's `latest` version alone does nothing to already-running
  containers.

## Cleanup

```bash
gcloud secrets versions disable 2 --secret=db-password
gcloud secrets versions disable 1 --secret=db-password
gcloud secrets delete db-password --quiet
kubectl delete secret db-password-k8s
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud secrets create --data-file=-` | Create a secret from stdin. |
| `gcloud secrets versions add --data-file=-` | Add a new version (rotate a value). |
| `gcloud secrets versions access latest` | Read the current value. |
| `gcloud secrets versions disable <n>` | Revoke access to an old version without deleting it. |
| `gcloud secrets add-iam-policy-binding --role=roles/secretmanager.secretAccessor` | Grant read-only access to a specific identity. |
| `gcloud run deploy --set-secrets="ENV=secret:version"` | Mount a secret as an env var in Cloud Run. |
| `gcloud secrets delete` | Permanently delete a secret and all its versions. |

## Exercise

Create a secret with one version, grant `roles/secretmanager.secretAccessor`
to a service account, and confirm access works with `gcloud secrets versions
access latest --secret=... --impersonate-service-account=...`. Rotate the
secret by adding a second version, disable the first, and confirm the old
version's `access` call now fails while `latest` returns the new value. If
you have a Cloud Run service handy, wire the secret in with
`--set-secrets` and confirm the container sees it as an environment variable.
