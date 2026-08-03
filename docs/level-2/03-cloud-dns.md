# 03 · Cloud DNS & Domains

Every service built so far has been reached by IP address or an auto-generated
`*.run.app` / `*.a.run.app` hostname. **Cloud DNS** is Google's managed,
authoritative DNS service — it lets you host the DNS records for your own
domain on Google's global anycast name-server network, so `app.example.com`
resolves to your load balancer instead of a raw IP nobody can remember.

## Managed zones: public vs. private

A **managed zone** is a container for the DNS records of one domain (and its
subdomains). Cloud DNS supports two kinds:

| | Public zone | Private zone |
|---|---|---|
| Visible to | The whole internet | Only VPCs you attach it to |
| Typical use | `example.com` for a public website | `internal.example.com` for services only reachable inside your VPC |
| Requires domain ownership | Yes — you must own/control the domain | No — any name works, it's never delegated publicly |

This module focuses on a public zone, since that's what fronts a live web app.

## Create a managed zone

```bash
gcloud services enable dns.googleapis.com

gcloud dns managed-zones create example-com \
  --dns-name="example.com." \
  --description="Public zone for example.com"
```

```bash
gcloud dns managed-zones describe example-com --format="value(nameServers)"
# ns-cloud-a1.googledomains.com.
# ns-cloud-a2.googledomains.com.
# ns-cloud-a3.googledomains.com.
# ns-cloud-a4.googledomains.com.
```

Cloud DNS assigns four name servers per zone. Nothing resolves publicly until
your domain registrar's NS records for `example.com` point at these four —
that delegation step happens outside GCP, at whichever registrar sold you the
domain (Google Domains, Namecheap, etc.).

## Add records

Records are added and removed through a **transaction**: you start one, stage
adds/removes, then execute it as a single atomic change.

```bash
gcloud dns record-sets transaction start --zone=example-com

gcloud dns record-sets transaction add 34.117.20.10 \
  --name="example.com." \
  --ttl=300 \
  --type=A \
  --zone=example-com

gcloud dns record-sets transaction add "www.example.com." \
  --name="www.example.com." \
  --ttl=300 \
  --type=CNAME \
  --zone=example-com

gcloud dns record-sets transaction execute --zone=example-com
```

```bash
gcloud dns record-sets list --zone=example-com
# NAME                TYPE  TTL   DATA
# example.com.        NS    21600 ns-cloud-a1.googledomains.com.,...
# example.com.        SOA   21600 ns-cloud-a1.googledomains.com. ...
# example.com.        A     300   34.117.20.10
# www.example.com.    CNAME 300   example.com.
```

A trailing dot (`example.com.`) means "this is a fully-qualified name, stop
here" — DNS tooling is strict about it, and a missing trailing dot on a
`--name` value is one of the most common Cloud DNS mistakes.

## Verifying propagation

```bash
dig example.com A +short
# 34.117.20.10

dig www.example.com CNAME +short
# example.com.

dig example.com NS +short
# ns-cloud-a1.googledomains.com.
# ns-cloud-a2.googledomains.com.
```

If `dig` against Google's own name servers works but the domain doesn't
resolve from your home network, the registrar-side NS delegation likely
hasn't propagated yet — that step is outside Cloud DNS's control and can take
anywhere from minutes to 48 hours depending on the registrar and old TTLs.

## Pointing a domain at a load balancer

For the global HTTP(S) load balancer built in [Module 02](02-autoscaling-load-balancing.md),
point the A record at the reserved global IP instead of a VM IP:

```bash
gcloud compute addresses describe web-lb-ip --global --format="value(address)"
# 34.117.20.10

gcloud dns record-sets transaction start --zone=example-com
gcloud dns record-sets transaction add 34.117.20.10 \
  --name="example.com." --ttl=300 --type=A --zone=example-com
gcloud dns record-sets transaction execute --zone=example-com
```

For Cloud Run, use a `CNAME` to the domain-mapping hostname Google gives you
after running `gcloud run domain-mappings create`, rather than a hardcoded IP
— Cloud Run's edge IPs are not guaranteed stable.

## DNSSEC

DNSSEC adds cryptographic signatures to DNS responses, preventing cache
poisoning / spoofed answers. Cloud DNS supports it per zone:

```bash
gcloud dns managed-zones update example-com --dnssec-state=on

gcloud dns dns-keys list --zone=example-com
```

Enabling DNSSEC produces a **DS record** that must also be published at the
registrar — enabling it in Cloud DNS alone does nothing until the registrar
side is configured too, and the two must be kept in sync if you ever rotate
keys.

## Terraform equivalent

```hcl
resource "google_dns_managed_zone" "example_com" {
  name     = "example-com"
  dns_name = "example.com."
}

resource "google_dns_record_set" "root_a" {
  name         = google_dns_managed_zone.example_com.dns_name
  managed_zone = google_dns_managed_zone.example_com.name
  type         = "A"
  ttl          = 300
  rrdatas      = ["34.117.20.10"]
}

resource "google_dns_record_set" "www_cname" {
  name         = "www.${google_dns_managed_zone.example_com.dns_name}"
  managed_zone = google_dns_managed_zone.example_com.name
  type         = "CNAME"
  ttl          = 300
  rrdatas      = [google_dns_managed_zone.example_com.dns_name]
}
```

## Gotchas

- **Delegation lives at the registrar, not in GCP.** Creating a Cloud DNS
  zone and its records does nothing publicly until the domain's registrar
  points its NS records at Cloud DNS's four name servers — a very common
  "why doesn't my domain work" dead end.
- **Trailing dots are load-bearing.** `--name` and `--dns-name` values are
  fully-qualified domain names and must end in `.`; forgetting it silently
  creates a record under the wrong (relative) name.
- **TTL controls how long old answers linger.** Lowering a record's TTL
  *before* a planned change (e.g. a migration cutover) so cached answers
  expire faster is a standard trick — raising TTL back afterward reduces
  query volume/cost.
- **DNSSEC needs registrar-side DS records too.** Turning on
  `--dnssec-state=on` in Cloud DNS is only half the job.
- **Private zones need explicit VPC attachment** (`--networks=`) — a private
  zone created without one resolves nowhere.

## Cleanup

```bash
gcloud dns record-sets transaction start --zone=example-com
gcloud dns record-sets transaction remove 34.117.20.10 \
  --name="example.com." --ttl=300 --type=A --zone=example-com
gcloud dns record-sets transaction remove "example.com." \
  --name="www.example.com." --ttl=300 --type=CNAME --zone=example-com
gcloud dns record-sets transaction execute --zone=example-com

gcloud dns managed-zones delete example-com
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud dns managed-zones create --dns-name=` | Create a public/private zone for a domain. |
| `gcloud dns managed-zones describe --format="value(nameServers)"` | Get the four name servers to configure at the registrar. |
| `gcloud dns record-sets transaction start/add/remove/execute` | Atomically change records in a zone. |
| `gcloud dns record-sets list` | List all records in a zone. |
| `gcloud dns managed-zones update --dnssec-state=on` | Enable DNSSEC signing. |
| `dig <name> <type> +short` | Query DNS directly to verify what's published. |

## Exercise

Create a managed zone for a domain you control (or a throwaway one for
practice), add an `A` record and a `www` `CNAME`, and verify both with `dig`
against Google's own name servers using `dig @ns-cloud-a1.googledomains.com
example.com A`. Then lower the `A` record's TTL to 60, change the IP, and
time how long it takes `dig` to return the new value. Clean up the zone
afterward.
