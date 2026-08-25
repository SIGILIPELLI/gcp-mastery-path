# 09 · Security Deep Dive (Security Command Center, VPC Service Controls)

This module covers organization-level security tooling: **Security Command
Center** for continuous threat/misconfiguration detection, and **VPC
Service Controls** for perimeter-based data exfiltration protection — a
different layer than IAM, which controls *who* can act; VPC-SC controls
*where data can flow*, regardless of who's authorized.

## Security Command Center (SCC)

SCC aggregates findings from GCP's built-in scanners (misconfigurations,
vulnerabilities, active threats) into one dashboard, org-wide.

```bash
gcloud scc findings list organizations/123456789012 \
  --filter="state=\"ACTIVE\" AND severity=\"HIGH\""
```

```bash
# NAME                                    CATEGORY                   SEVERITY  STATE
# .../findings/a1b2c3                     PUBLIC_BUCKET_ACL          HIGH      ACTIVE
# .../findings/d4e5f6                     OPEN_FIREWALL              HIGH      ACTIVE
```

```bash
gcloud scc findings describe organizations/123456789012 \
  --finding=a1b2c3 --source=SOURCE_ID
# category: PUBLIC_BUCKET_ACL
# resourceName: //storage.googleapis.com/orders-exports-bucket
# recommendation: Remove allUsers/allAuthenticatedUsers from bucket IAM policy
```

Once remediated, mark the finding to keep the dashboard signal-to-noise
ratio usable:

```bash
gcloud scc findings update organizations/123456789012 \
  --finding=a1b2c3 --source=SOURCE_ID --state=INACTIVE
```

**Gotcha — SCC tiers gate what you get.** The free Standard tier surfaces
a limited set of findings (mostly asset inventory and a subset of
misconfigurations); Premium tier adds threat detection (Event Threat
Detection, Container Threat Detection) and vulnerability scanning. Assuming
"SCC will catch it" without checking which tier is enabled at the org level
is a common gap — `gcloud scc settings describe organizations/123456789012`
shows the active tier.

## VPC Service Controls

A **service perimeter** wraps a set of projects and restricts API calls
(GCS, BigQuery, etc.) to only work from within trusted networks/identities
— even a compromised, fully-authorized credential can't exfiltrate data by
copying it to an outside project, because the perimeter blocks the API
call itself, not just the credential.

```bash
gcloud access-context-manager policies create \
  --organization=123456789012 \
  --title="corp-policy"

gcloud access-context-manager perimeters create prod-perimeter \
  --title="Production Data Perimeter" \
  --resources=projects/111111111111,projects/222222222222 \
  --restricted-services=storage.googleapis.com,bigquery.googleapis.com \
  --policy=POLICY_ID
```

```bash
gcloud access-context-manager perimeters describe prod-perimeter --policy=POLICY_ID
# title: Production Data Perimeter
# status:
#   resources: [projects/111111111111, projects/222222222222]
#   restrictedServices: [storage.googleapis.com, bigquery.googleapis.com]
```

Now, a `gsutil cp` attempting to copy from a bucket inside the perimeter to
a bucket in a project *outside* it fails — `403 Request is prohibited by
organization's policy` — regardless of the caller's IAM roles, because the
call crosses the perimeter boundary.

**Gotcha — perimeters block legitimate cross-project workflows too, by
design.** A CI pipeline in an unrelated tooling project trying to read a
protected BigQuery dataset breaks the same way a malicious exfiltration
attempt would. The fix is an explicit **ingress/egress rule** allowing that
specific caller identity and service combination — not disabling the
perimeter:

```bash
gcloud access-context-manager perimeters update prod-perimeter \
  --policy=POLICY_ID \
  --add-egress-policies=egress-ci-readonly.yaml
```

```yaml
# egress-ci-readonly.yaml
- egressFrom:
    identities:
      - serviceAccount:ci-sa@tooling-project.iam.gserviceaccount.com
  egressTo:
    operations:
      - serviceName: bigquery.googleapis.com
        methodSelectors:
          - method: "*"
    resources:
      - projects/111111111111
```

**Gotcha — perimeters have a dry-run mode, and skipping it is how outages
happen.** Always deploy new/changed perimeters with `--dry-run` first;
dry-run logs what *would* be blocked without actually blocking it, so you
can find missing ingress/egress rules before they break production traffic:

```bash
gcloud access-context-manager perimeters dry-run enforce prod-perimeter --policy=POLICY_ID
gcloud logging read 'protoPayload.metadata.dryRun=true' --limit=20
```

## Binary Authorization (brief)

Complementary control for the deploy path: only allow container images
signed by a trusted attestor to run on GKE/Cloud Run.

```bash
gcloud container binauthz policy import policy.yaml

gcloud container binauthz attestations sign-and-create \
  --artifact-url=us-central1-docker.pkg.dev/my-project/app/orders@sha256:abc... \
  --attestor=prod-attestor \
  --attestor-project=my-project \
  --keyversion=projects/my-project/locations/us-central1/keyRings/binauthz/cryptoKeys/attestor-key/cryptoKeyVersions/1
```

**Gotcha — Binary Authorization checks at deploy time, not build time.** An
unsigned image can still sit in Artifact Registry indefinitely; the block
only triggers on `gcloud run deploy` / `kubectl apply` against a
Binary-Authorization-enforced cluster or service. Don't mistake "the image
exists in the registry" for "it passed policy."

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud scc findings list` | List active security findings org-wide. |
| `gcloud scc settings describe` | Check which SCC tier (Standard/Premium) is active. |
| `gcloud access-context-manager perimeters create` | Wrap projects in a VPC-SC perimeter. |
| `--add-egress-policies` / `--add-ingress-policies` | Punch a scoped exception through a perimeter. |
| `perimeters dry-run enforce` | Test a perimeter's impact before enforcing it. |
| `gcloud container binauthz policy import` | Require signed/attested images at deploy time. |

## Exercise

Design (command sequence, no live resources needed) a VPC Service Controls
perimeter around two projects restricting `storage.googleapis.com`. Write
the dry-run command you'd run first, describe what you'd look for in the
dry-run logs, and write one egress rule allowing a specific CI service
account read-only access to a bucket inside the perimeter from a project
outside it.
