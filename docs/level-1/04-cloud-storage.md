# 04 · Cloud Storage

**Cloud Storage** is GCP's object storage service — durable, scalable
storage for files ("objects") grouped into **buckets**, accessed over HTTP(S)
or the `gsutil`/`gcloud storage` CLIs. This module covers creating buckets,
managing objects, storage classes, and hosting a static website.

## Buckets and global uniqueness

Bucket names are **globally unique across all of Cloud Storage**, not just
your project — like domain names. Pick something namespaced, e.g.
`gcp-mastery-path-123-assets`.

```bash
# Create a bucket (uniform bucket-level access is the modern default)
gcloud storage buckets create gs://gcp-mastery-path-123-assets \
  --location=us-central1 \
  --uniform-bucket-level-access
```

`--location` can be a single region (`us-central1`), a dual-region, or a
multi-region (`US`) — multi-region costs more per GB but serves from
whichever region is closest to the requester and survives a full regional
outage.

## Uploading, listing, downloading objects

```bash
# Upload
gcloud storage cp ./report.pdf gs://gcp-mastery-path-123-assets/reports/report.pdf

# Upload a whole directory
gcloud storage cp -r ./site gs://gcp-mastery-path-123-assets/site

# List
gcloud storage ls gs://gcp-mastery-path-123-assets/
gcloud storage ls -l gs://gcp-mastery-path-123-assets/reports/

# Download
gcloud storage cp gs://gcp-mastery-path-123-assets/reports/report.pdf ./report.pdf

# Delete an object
gcloud storage rm gs://gcp-mastery-path-123-assets/reports/report.pdf

# Sync a local directory to a bucket (only uploads changed files)
gcloud storage rsync ./site gs://gcp-mastery-path-123-assets/site
```

!!! note "gsutil vs. gcloud storage"
    `gsutil` is the original CLI for Cloud Storage and still works
    everywhere you'll see it in older docs and scripts (`gsutil cp`,
    `gsutil ls`, `gsutil rsync`). `gcloud storage` is the newer, faster,
    actively-developed replacement with near-identical syntax. Prefer
    `gcloud storage` for new scripts; recognize `gsutil` when you meet it.

## Storage classes

| Class | Min. storage duration | Typical use |
|---|---|---|
| **Standard** | None | Frequently accessed ("hot") data, website assets |
| **Nearline** | 30 days | Data accessed roughly monthly (backups) |
| **Coldline** | 90 days | Data accessed roughly quarterly (archival tiers) |
| **Archive** | 365 days | Long-term retention, disaster recovery, rarely read |

Lower classes cost less per GB stored but more per GB retrieved, and charge
an early-deletion penalty if removed before the minimum duration. Set a
class per object or per bucket default:

```bash
gcloud storage buckets create gs://gcp-mastery-path-123-backups \
  --location=us-central1 \
  --default-storage-class=NEARLINE
```

Automate transitions with a **lifecycle rule** — e.g. move objects to
Coldline after 90 days, then delete after 365:

```bash
cat > lifecycle.json << 'EOF'
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
      "condition": {"age": 90}
    },
    {
      "action": {"type": "Delete"},
      "condition": {"age": 365}
    }
  ]
}
EOF

gcloud storage buckets update gs://gcp-mastery-path-123-backups \
  --lifecycle-file=lifecycle.json
```

## Access control

With **uniform bucket-level access** (the recommended default), permissions
are granted only via IAM, applied consistently to every object in the bucket
— no legacy per-object ACLs to reason about.

```bash
# Let anyone on the internet read objects in this bucket
gcloud storage buckets add-iam-policy-binding gs://gcp-mastery-path-123-assets \
  --member=allUsers \
  --role=roles/storage.objectViewer

# Grant one teammate the ability to upload/overwrite objects
gcloud storage buckets add-iam-policy-binding gs://gcp-mastery-path-123-assets \
  --member=user:teammate@example.com \
  --role=roles/storage.objectAdmin
```

!!! warning "`allUsers` means the public internet"
    Only grant `allUsers` read access on buckets meant to be public, like
    static website assets. Never grant `allUsers` write access.

## Static website hosting

Cloud Storage can serve a bucket's contents directly as a static website —
no server required.

```bash
# Make the bucket name match your intended site, upload content, make it public
gcloud storage cp -r ./site/* gs://gcp-mastery-path-123-assets/

gcloud storage buckets add-iam-policy-binding gs://gcp-mastery-path-123-assets \
  --member=allUsers \
  --role=roles/storage.objectViewer

# Configure the index/error pages
gcloud storage buckets update gs://gcp-mastery-path-123-assets \
  --web-main-page-suffix=index.html \
  --web-error-page=404.html
```

The bucket is now reachable at
`https://storage.googleapis.com/gcp-mastery-path-123-assets/index.html`. For
a custom domain over HTTPS with a clean URL, you'd front the bucket with a
global external Application Load Balancer and a managed SSL certificate —
covered later in this series alongside Cloud CDN.

## Cleanup

```bash
gcloud storage rm --recursive gs://gcp-mastery-path-123-assets
gcloud storage buckets delete gs://gcp-mastery-path-123-assets --quiet
gcloud storage buckets delete gs://gcp-mastery-path-123-backups --quiet
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud storage buckets create gs://<name>` | Create a bucket. |
| `gcloud storage cp` | Upload/download/copy objects. |
| `gcloud storage rsync` | Sync a local directory and a bucket path. |
| `gcloud storage ls` | List buckets or objects. |
| `gcloud storage rm` | Delete objects (`--recursive` for a whole prefix). |
| `gcloud storage buckets update --lifecycle-file=` | Set an automated class-transition/expiry policy. |
| `gcloud storage buckets add-iam-policy-binding` | Grant access to a bucket. |
| `gcloud storage buckets update --web-main-page-suffix=` | Enable static website hosting on a bucket. |
| `gcloud storage buckets delete` | Permanently delete an (empty) bucket. |

## Exercise

Create a bucket, upload a small `index.html` you write yourself, make the
bucket's objects public, and enable static website hosting with
`index.html` as the main page. Fetch
`https://storage.googleapis.com/<your-bucket>/index.html` with `curl` to
confirm it serves your content, then delete the object and the bucket.
