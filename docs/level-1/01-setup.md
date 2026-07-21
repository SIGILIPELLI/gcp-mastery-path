# 01 · Setup & gcloud CLI

Every GCP lesson in this level assumes you have a Google Cloud account, a
project to work in, and the `gcloud` command-line tool installed and
authenticated. This module gets all three in place, and explains the billing
model so you can use the free tier with confidence instead of anxiety.

## Create a Google Cloud account

Go to [cloud.google.com](https://cloud.google.com/) and sign in with any
Google account (or create one). New accounts get:

- **A $300 free trial credit**, usable for 90 days across almost any GCP
  service.
- **"Always free" tier** limits on specific products (e.g. 1 `e2-micro`
  Compute Engine instance per month in certain US regions, 5 GB of
  Cloud Storage, 2 million Cloud Functions invocations/month) that don't
  expire when the trial ends — as long as you stay under the monthly
  threshold, they're free forever.

During signup Google **requires a credit card** even for the free trial —
this is for identity verification and to let you continue past the trial if
you choose. Trial accounts are **not auto-charged** when the $300 credit or
90 days runs out; GCP pauses your resources and asks you to explicitly
upgrade to a paid account. Still, this course treats teardown as mandatory
practice (see the cheat-sheet and every lesson's cleanup section) so you
build the right habits from day one.

## Create your first project

Everything in GCP lives inside a **project** — the unit of billing, IAM, and
API enablement. Create one from the [console](https://console.cloud.google.com/)
(top-left project picker → "New Project"), or from the CLI once it's
installed (below). Project IDs are globally unique, lowercase, and immutable
once created — pick something like `gcp-mastery-path-123`.

## Install the gcloud CLI

=== "macOS"

    ```bash
    brew install --cask google-cloud-sdk
    ```

=== "Linux (Debian/Ubuntu)"

    ```bash
    sudo apt-get install apt-transport-https ca-certificates gnupg curl
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    echo "deb https://packages.cloud.google.com/apt cloud-sdk main" | \
      sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
    sudo apt-get update && sudo apt-get install google-cloud-cli
    ```

=== "Windows"

    Download and run the installer from
    [cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install),
    or use the interactive installer inside **Cloud Shell** in the console —
    no local install needed at all if you'd rather work there first.

Verify the install:

```bash
gcloud --version
# Google Cloud SDK 4xx.0.0
# bq 2.x.x
# core 2026.xx.xx
# gsutil 5.xx
```

!!! tip "Don't want to install anything yet?"
    [Cloud Shell](https://cloud.google.com/shell) gives you a free, temporary
    VM in your browser with `gcloud`, `gsutil`, `bq`, `kubectl`, Terraform,
    and an editor already installed and pre-authenticated to your account.
    Every command in this course works there unmodified.

## Authenticate and initialize

```bash
# Opens a browser to log in with your Google account
gcloud auth login

# Interactive setup: pick account, project, and default region/zone
gcloud init
```

`gcloud init` walks you through:

1. Choosing (or re-using) an authenticated account.
2. Selecting or creating a default project.
3. Optionally setting a default **compute region and zone** — this saves you
   from passing `--zone`/`--region` on every later command.

Set these explicitly at any time:

```bash
gcloud config set project gcp-mastery-path-123
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
```

Check your current configuration:

```bash
gcloud config list
# [core]
# account = you@example.com
# project = gcp-mastery-path-123
# [compute]
# region = us-central1
# zone = us-central1-a
```

## Application Default Credentials

Separately from `gcloud auth login` (which authenticates the CLI itself),
client libraries and tools like Terraform look for **Application Default
Credentials (ADC)**:

```bash
gcloud auth application-default login
```

This writes credentials to a well-known local path that the Cloud client
libraries (Python, Node, Go, etc.) and Terraform's Google provider pick up
automatically — you'll use this in the Terraform module later in this level.

## Enable billing and APIs

A project needs an active **billing account** linked before most services
will work (the free trial credit is attached to a billing account, so this
still applies even if nothing gets charged).

```bash
# List your billing accounts
gcloud billing accounts list

# Link one to your project
gcloud billing projects link gcp-mastery-path-123 \
  --billing-account=XXXXXX-XXXXXX-XXXXXX
```

GCP services are exposed as individually-enabled APIs. Enable the ones this
level uses now, so later lessons don't stop to ask:

```bash
gcloud services enable \
  compute.googleapis.com \
  storage.googleapis.com \
  sqladmin.googleapis.com \
  run.googleapis.com \
  cloudfunctions.googleapis.com \
  logging.googleapis.com \
  monitoring.googleapis.com
```

## Projects, folders, and organizations

| Concept | Scope |
|---|---|
| **Project** | The base container for resources, billing, and IAM. Everything you create lives in exactly one project. |
| **Folder** | Optional grouping of projects (and other folders) — used to apply policy to a group of projects at once. |
| **Organization** | The root node, tied to a Google Workspace/Cloud Identity domain. Personal Gmail accounts skip this — your projects are "No organization." |
| **Billing account** | Pays for one or more projects. A project can have zero (disabled) or one linked billing account. |

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud init` | Interactive first-time setup (account, project, region/zone). |
| `gcloud auth login` | Authenticate the CLI to a Google account. |
| `gcloud auth application-default login` | Set up ADC for client libraries and Terraform. |
| `gcloud config set project <id>` | Set the active default project. |
| `gcloud config list` | Show current CLI configuration. |
| `gcloud projects create <id>` | Create a new project from the CLI. |
| `gcloud billing projects link <id> --billing-account=<acct>` | Attach billing to a project. |
| `gcloud services enable <api>` | Turn on a GCP API for the current project. |
| `gcloud services list --enabled` | List APIs enabled on the current project. |

## Exercise

Create a new project called `gcp-mastery-path-<yourname>`, link a billing
account to it, install `gcloud` (or open Cloud Shell), run `gcloud init`
against the new project, and enable the `compute.googleapis.com` and
`storage.googleapis.com` APIs. Confirm with `gcloud config list` and
`gcloud services list --enabled` that everything points at the right project.
