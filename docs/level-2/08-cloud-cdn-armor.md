# 08 · Cloud CDN & Cloud Armor

The global HTTP(S) load balancer built in [Module 02](02-autoscaling-load-balancing.md)
sends every request all the way to a backend instance. Two add-ons attach
directly to that same load balancer without changing your application code:
**Cloud CDN** caches responses at Google's edge so repeat requests never
reach your backend at all, and **Cloud Armor** filters requests *before*
they reach your backend, blocking abuse and attacks at the edge.

## Cloud CDN: enable on a backend service

CDN attaches to the backend service from Module 02's load-balancing chain —
it's a property of the backend, not a separate resource:

```bash
gcloud compute backend-services update web-backend \
  --global \
  --enable-cdn \
  --cache-mode=CACHE_ALL_STATIC
```

```bash
gcloud compute backend-services describe web-backend --global \
  --format="value(enableCDN,cdnPolicy.cacheMode)"
# True  CACHE_ALL_STATIC
```

## Cache modes

| Mode | Behavior |
|---|---|
| `USE_ORIGIN_HEADERS` | Respect the origin's own `Cache-Control` headers exactly |
| `CACHE_ALL_STATIC` | Cache common static file types automatically, regardless of headers |
| `FORCE_CACHE_ALL` | Cache everything, including responses that say not to — dangerous for dynamic/authenticated content |

`FORCE_CACHE_ALL` will happily cache a response containing one user's private
data and serve it to the next unrelated visitor if the URL is shared and the
cache key doesn't account for the difference — reserve it for genuinely
static, public assets only.

## Cache key and invalidation

```bash
gcloud compute backend-services update web-backend \
  --global \
  --cache-key-include-protocol \
  --cache-key-include-host \
  --no-cache-key-include-query-string
```

Excluding the query string from the cache key means `/product?ref=email` and
`/product?ref=ads` are treated as the *same* cached object — correct for
tracking parameters that don't change the response, wrong if a query
parameter actually changes the content served.

```bash
# Force-refresh cached content immediately, instead of waiting for TTL expiry
gcloud compute url-maps invalidate-cdn-cache web-url-map \
  --path="/static/*"
```

```text
Invalidating path /static/* on url-map [web-url-map]...done.
```

## Checking cache performance

```bash
curl -sD - -o /dev/null http://34.117.20.10/static/logo.png | grep -i cache
# X-Cache: HIT
# Age: 42
```

`Age` is seconds since the object was fetched from the origin — a growing
`Age` with `X-Cache: HIT` confirms the CDN is actually serving from cache
rather than re-fetching every request.

## Cloud Armor: a basic security policy

Cloud Armor security policies attach to a backend service the same way CDN
does, and evaluate rules **before** a request reaches any backend:

```bash
gcloud compute security-policies create web-security-policy \
  --description="Baseline protections for web-backend"
```

```bash
# Block a known-bad IP range outright
gcloud compute security-policies rules create 1000 \
  --security-policy=web-security-policy \
  --src-ip-ranges="198.51.100.0/24" \
  --action=deny-403

# Allow everything else (implicit default, made explicit here)
gcloud compute security-policies rules update 2147483647 \
  --security-policy=web-security-policy \
  --action=allow
```

Rules are evaluated in **priority order** (lower number = higher priority,
evaluated first) — `2147483647` is the reserved priority for the policy's
default catch-all rule.

## Rate limiting

```bash
gcloud compute security-policies rules create 900 \
  --security-policy=web-security-policy \
  --expression="true" \
  --action=rate-based-ban \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=600 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP
```

A client exceeding 100 requests/minute gets `429`s and then a 10-minute ban —
`--enforce-on-key=IP` counts per source IP, which is the common default but
can also key on a header (e.g. an API key) for per-client limits behind a
shared NAT/proxy.

## Preconfigured WAF rules

Cloud Armor ships with preconfigured rule sets for common attack patterns
(OWASP Top 10-style), so you don't have to author SQL-injection/XSS
detection from scratch:

```bash
gcloud compute security-policies rules create 800 \
  --security-policy=web-security-policy \
  --expression="evaluatePreconfiguredExpr('sqli-stable')" \
  --action=deny-403
```

## Attach the policy to the backend

```bash
gcloud compute backend-services update web-backend \
  --global \
  --security-policy=web-security-policy
```

```bash
gcloud compute backend-services describe web-backend --global \
  --format="value(securityPolicy)"
# https://www.googleapis.com/compute/v1/projects/gcp-mastery-path-123/global/securityPolicies/web-security-policy
```

## Terraform equivalent

```hcl
resource "google_compute_security_policy" "web" {
  name = "web-security-policy"

  rule {
    action   = "deny(403)"
    priority = 1000
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["198.51.100.0/24"]
      }
    }
  }

  rule {
    action   = "allow"
    priority = 2147483647
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
  }
}

resource "google_compute_backend_service" "web" {
  name            = "web-backend"
  enable_cdn      = true
  security_policy = google_compute_security_policy.web.id
  # ... health_checks, backend blocks from Module 02
}
```

## Gotchas

- **CDN can cache the wrong content if the cache key is too loose.**
  Excluding query strings/headers from the cache key is a common source of
  "why is one user seeing another user's data" bugs — only exclude what you
  are certain doesn't affect the response.
- **`FORCE_CACHE_ALL` is a footgun for anything not genuinely public and
  static.** Default to `CACHE_ALL_STATIC` or `USE_ORIGIN_HEADERS` instead.
- **Cache invalidation isn't instant globally** — `invalidate-cdn-cache`
  can take a short time to propagate across all edge locations; don't assume
  the very next request everywhere reflects it immediately.
- **Cloud Armor rule priority order matters, and a `deny` rule with a
  numerically higher priority than an `allow` rule for the same traffic
  loses** — rules are evaluated lowest-priority-number-first, and the first
  match wins.
- **Rate-based rules count *after* the request reaches Google's edge**, so
  they protect your backend's capacity but don't reduce Cloud Armor's own
  evaluation load — a true DDoS at massive scale needs Google's automatic
  network-layer protections underneath Cloud Armor, not Cloud Armor alone.

## Cleanup

```bash
gcloud compute backend-services update web-backend --global --no-security-policy
gcloud compute security-policies delete web-security-policy --quiet
gcloud compute backend-services update web-backend --global --no-enable-cdn
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `gcloud compute backend-services update --enable-cdn` | Turn on CDN caching for a backend. |
| `gcloud compute url-maps invalidate-cdn-cache --path=` | Force-expire cached objects by path. |
| `gcloud compute security-policies create` | Create a Cloud Armor policy. |
| `gcloud compute security-policies rules create <priority>` | Add an allow/deny/rate-limit rule. |
| `evaluatePreconfiguredExpr('sqli-stable')` | Use Google's built-in WAF rule sets in an expression. |
| `gcloud compute backend-services update --security-policy=` | Attach a Cloud Armor policy to a backend. |

## Exercise

Enable CDN on the load balancer from Module 02 with `CACHE_ALL_STATIC`, `curl`
a static asset twice, and confirm the second response shows `X-Cache: HIT`
with a growing `Age`. Then create a Cloud Armor policy with a rate-based rule
(100 req/min) and attach it to the same backend; use a loop of `curl`
requests to exceed the threshold and confirm you start getting `429`
responses, then wait out the ban duration and confirm access is restored.
