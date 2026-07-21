# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: get comfortable with the `gcloud` CLI and GCP console, understand
identity and security basics, and stand up (and safely tear down) a small
end-to-end web app using free-tier-friendly services.

## Modules

1. [Setup & gcloud CLI](01-setup.md)
2. [IAM & Security Basics](02-iam-security.md)
3. [Compute Engine](03-compute-engine.md)
4. [Cloud Storage](04-cloud-storage.md)
5. [VPC Networking Basics](05-vpc-networking.md)
6. [Cloud SQL](06-cloud-sql.md)
7. [Cloud Functions & Cloud Run](07-serverless-run-functions.md)
8. [Cloud Monitoring & Logging](08-monitoring-logging.md)
9. [Infrastructure as Code (Terraform on GCP)](09-terraform-intro.md)
10. [Capstone Project](10-capstone-project.md)

By the end of this level you'll be able to create and secure a GCP project,
launch a VM, store and serve objects from a bucket, wire up basic networking,
run a managed database, deploy a container or function, watch your resources
with monitoring and logs, describe infrastructure as code, and — just as
important on a free-tier account — tear everything down cleanly afterward.

!!! warning "This level uses real GCP resources"
    Every lesson here creates real resources in a real GCP project. GCP's
    [free tier](https://cloud.google.com/free) covers a generous set of
    usage, but it is not unlimited. Follow the cleanup commands at the end of
    each lesson, and especially the teardown section in the capstone, so you
    don't leave anything running that could incur charges.
