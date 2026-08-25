# 04 · Advanced IAM (Org Policies, Custom Roles)

Levels 1-2 used predefined roles like `roles/storage.objectViewer`. At
organization scale you also need **custom roles** for least-privilege
grants predefined roles don't match, and **Organization Policies** to set
guardrails that apply across every project underneath an org or folder,
regardless of who holds Owner on any individual project.

## Custom roles

```bash
gcloud iam roles create orderServiceDeployer \
  --project=my-project \
  --title="Order Service Deployer" \
  --description="Deploy and view logs for the order service, nothing else" \
  --permissions=run.services.get,run.services.update,run.revisions.list,logging.logEntries.list \
  --stage=GA
```

```bash
gcloud iam roles describe orderServiceDeployer --project=my-project
# name: projects/my-project/roles/orderServiceDeployer
# includedPermissions:
# - logging.logEntries.list
# - run.revisions.list
# - run.services.get
# - run.services.update
# stage: GA
```

Grant it exactly like a predefined role:

```bash
gcloud projects add-iam-policy-binding my-project \
  --member="user:dev@example.com" \
  --role="projects/my-project/roles/orderServiceDeployer"
```

**Gotcha — custom roles don't auto-update with new GCP permissions.**
Predefined roles get new permissions added by Google as services evolve;
a custom role is a frozen permission list you own and must maintain
yourself. Review custom roles periodically against
`gcloud iam roles describe roles/run.developer` (or whichever predefined
role you modeled it on) to catch drift.

**Gotcha — testing permission sets before granting.** Use
`gcloud iam roles create --stage=TESTING` for a role you're still tuning; a
`TESTING` role behaves identically for grants but is excluded from
recommender suggestions and flagged as provisional in audit tooling, so
reviewers know not to treat it as final.

## Organization Policies

Org Policies are **constraints**, not permissions — they restrict what's
*possible*, even for a project Owner. They apply at org, folder, or project
level and inherit downward.

```bash
# List all available constraints
gcloud org-policies list --organization=123456789012 | head -5

# Restrict VM external IPs org-wide
gcloud org-policies set-policy external-ip-policy.yaml --organization=123456789012
```

```yaml
# external-ip-policy.yaml
name: organizations/123456789012/policies/compute.vmExternalIpAccess
spec:
  rules:
    - denyAll: true
```

```bash
gcloud org-policies describe compute.vmExternalIpAccess \
  --organization=123456789012
# spec:
#   rules:
#   - denyAll: true
```

A VM creation that would assign an external IP now fails at admission time
— `ERROR: (gcloud.compute.instances.create) ... Constraint
constraints/compute.vmExternalIpAccess violated` — no matter who runs it or
what project-level IAM they hold.

**Gotcha — inheritance can be overridden per-folder, deliberately or by
accident.** A child folder/project can set its own policy that overrides an
inherited one (unless the parent constraint is marked non-overridable at
the org level). This is intentional for exceptions (e.g., a
public-demos folder that needs external IPs), but it means "we set an org
policy" doesn't guarantee it holds everywhere — check effective policy per
resource with `gcloud org-policies describe CONSTRAINT --project=X` and
compare against the org-level setting.

## Common constraints worth knowing

| Constraint | Effect |
|---|---|
| `compute.vmExternalIpAccess` | Block/allow list which VMs may have external IPs. |
| `iam.disableServiceAccountKeyCreation` | Forbid creating downloadable SA JSON keys org-wide. |
| `compute.requireOsLogin` | Force OS Login instead of SSH metadata keys for VM access. |
| `compute.restrictVpcPeering` | Limit which VPCs/projects may peer with each other. |
| `gcp.resourceLocations` | Restrict which regions resources may be created in (data residency). |
| `iam.allowedPolicyMemberDomains` | Restrict IAM grants to specific identity domains (block gmail.com principals in a corp org). |

## IAM Conditions recap and denial policies

Level 1-2 covered basic role bindings. **IAM Deny policies** are the
complement to Allow bindings — they take precedence regardless of any Allow
grant, useful for hard org-wide exclusions like "nobody may delete
production BigQuery datasets, even Owners":

```bash
gcloud iam deny-policies create no-prod-dataset-delete \
  --organization=123456789012 \
  --policy-file=deny-policy.yaml
```

```yaml
# deny-policy.yaml
displayName: "no-prod-dataset-delete"
rules:
  - denyRule:
      deniedPrincipals:
        - "principalSet://goog/public:all"
      deniedPermissions:
        - "bigquery.datasets.delete"
      denialCondition:
        expression: 'resource.name.startsWith("projects/prod-")'
```

**Gotcha — Deny policies evaluate before Allow, always.** If a Deny policy
matches, no Allow binding can override it, including an Owner role — the
only fix is to edit or delete the Deny policy itself. This makes Deny
policies powerful and also easy to lock yourself out with; test in a
non-prod org node first.

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud iam roles create` | Define a custom role with an exact permission set. |
| `gcloud org-policies set-policy` | Apply a constraint at org/folder/project scope. |
| `gcloud org-policies describe` | Check the effective policy for a constraint on a resource. |
| `gcloud iam deny-policies create` | Create a hard deny that overrides any Allow binding. |
| `--stage=TESTING` | Mark a custom role as provisional during tuning. |

## Exercise

Create a custom role with exactly the four permissions needed to view and
restart a Cloud Run revision (no update/delete). Grant it to a test user.
Separately, write and apply an org-policy YAML that denies VM external IPs
at your test project (project-level override is fine without an actual
org), attempt to create a VM with `--no-address` omitted, and confirm the
constraint violation error. Then describe the effective policy to prove
it's active.
