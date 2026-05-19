---
title: "Setting Up Google Cloud Load Balancer with SSL and CDN"
author: " "
date: 2026-05-04
draft: false
tags: ["hugo", "gcp", "load-balancer", "ssl", "cdn", "devops"]
categories: ["Tutorial"]
description: "Step by step guide to setting up a Global Cloud Load Balancer with SSL certificate, Cloud CDN, and HTTP to HTTPS redirect for a Hugo static site on GCS."
showToc: true
---

This is Part 8 of a series on building and deploying a personal resume
site using Hugo and Google Cloud Platform. In this post we cover setting
up a Global Cloud Load Balancer with SSL, CDN, and automatic HTTP to
HTTPS redirection — the infrastructure that serves `https://example.com`
to visitors worldwide.

---

## Why a Load Balancer for a Static Site?

A Load Balancer might seem overkill for a resume site, but it solves
several problems that GCS static hosting cannot handle alone:

```
Without Load Balancer:
    ❌ No HTTPS on custom domain
    ❌ Directory routing broken (/experience/ → NoSuchKey)
    ❌ No CDN
    ❌ Raw GCS URL (storage.googleapis.com/example.com)

With Load Balancer:
    ✅ HTTPS with Google-managed SSL certificate
    ✅ Clean directory routing
    ✅ Cloud CDN enabled
    ✅ Custom domain (example.com)
```

> [!NOTE] There are easy alternatives to implementing LB but this is to complete CR challenge.

---

## Architecture Overview

```
https://example.com
        ↓
Static IP (34.x.x.x)
        ↓
Forwarding Rule (port 443)
        ↓
HTTPS Proxy (terminates SSL)
        ↓
URL Map (routing decisions)
        ↓
Backend Bucket (pointer to GCS)
        ↓
Cloud CDN (cache check)
        ↓
gs://example.com (Hugo files)
```

HTTP traffic on port 80 is automatically redirected to HTTPS:

```
http://example.com
        ↓
Forwarding Rule (port 80)
        ↓
HTTP Proxy
        ↓
Redirect URL Map
        ↓
301 → https://example.com
```

---

## Components Created

### 1. Static IP Address

```bash
gcloud compute addresses create my-resume-ip \
    --network-tier=PREMIUM \
    --ip-version=IPV4 \
    --global \
    --project=my-resume-123456
```

A static IP never changes — essential for DNS configuration. Without
it the IP could change and break your DNS records.

**Why PREMIUM tier?**
- Global Load Balancer requires PREMIUM network tier
- PREMIUM routes traffic through Google's global backbone
- Required for Cloud CDN
- STANDARD tier only supports regional load balancers which
  cannot use GCS as a backend

### 2. Backend Bucket

```bash
gcloud compute backend-buckets create my-resume-backend \
    --gcs-bucket-name=example.com \
    --enable-cdn \
    --project=my-resume-123456
```

A Backend Bucket is a **Load Balancer configuration object** — not
a storage container. It is a pointer that tells the Load Balancer
where your GCS bucket is. It contains no files and runs no code.

```
GCS Bucket     = actual storage (files live here)
Backend Bucket = connector (tells LB about GCS bucket)
```

**CDN enabled** — Cloud CDN caches your site at GCP edge nodes
near visitors. Even for US-only traffic this matters:

```
Without CDN: Boston visitor → Iowa (1,300 miles) → 50-100ms
With CDN:    Boston visitor → New York (200 miles) → 5-10ms
```

### 3. URL Map

```bash
gcloud compute url-maps create my-resume-urlmap \
    --default-backend-bucket=my-resume-backend \
    --project=my-resume-123456
```

A URL Map is a **routing table** for the Load Balancer. It decides
which backend receives each request based on URL path and hostname.

Our URL Map is simple — send all traffic to one backend:

```
ANY request to example.com/* → my-resume-backend
```

For complex sites URL Maps can route to multiple backends:

```
example.com/      → GCS bucket (static resume)
example.com/api/* → Cloud Run (API service)
example.com/blog/ → different GCS bucket (blog)
```

### 4. SSL Certificate

```bash
gcloud compute ssl-certificates create my-resume-cert \
    --domains=example.com \
    --global \
    --project=my-resume-123456
```

A **Google-managed certificate** — automatically provisioned and
renewed by Google. No manual renewal ever needed.

Certificate status starts as `PROVISIONING` and becomes `ACTIVE`
only after DNS points to the Load Balancer IP. Google verifies
domain ownership by checking DNS.

### 5. HTTPS Proxy

```bash
gcloud compute target-https-proxies create my-resume-https-proxy \
    --ssl-certificates=my-resume-cert \
    --url-map=my-resume-urlmap \
    --global \
    --project=my-resume-123456
```

The HTTPS Proxy is a **reverse proxy** — it sits between the visitor
and your GCS bucket acting on behalf of your server. It has two jobs:

```
Job 1 (Security):
    → Holds SSL certificate
    → Proves identity to browsers
    → Terminates encrypted HTTPS connection
    → Decrypts incoming requests

Job 2 (Routing):
    → Passes decrypted request to URL Map
    → Gets routing decision
    → Forwards to correct backend
```

### 6. HTTP to HTTPS Redirect

```bash
# Redirect URL Map
gcloud compute url-maps import my-resume-http-redirect \
    --source=/tmp/http-redirect.yaml \
    --global \
    --project=my-resume-123456
```

```yaml
# /tmp/http-redirect.yaml
name: my-resume-http-redirect
defaultUrlRedirect:
  redirectResponseCode: MOVED_PERMANENTLY_DEFAULT
  httpsRedirect: true
```

This URL Map sends **no content** — it only returns a 301 redirect:

```
http://example.com/experience/
        ↓
301 Moved Permanently
Location: https://example.com/experience/
        ↓
Browser retries on HTTPS ✅
```

`httpsRedirect: true` changes only the scheme (`http` → `https`)
keeping the full path, query string, and fragment identical.

### 7. Forwarding Rules

```bash
# HTTPS - port 443
gcloud compute forwarding-rules create my-resume-https-rule \
    --address=my-resume-ip \
    --target-https-proxy=my-resume-https-proxy \
    --ports=443 \
    --global \
    --project=my-resume-123456

# HTTP - port 80
gcloud compute forwarding-rules create my-resume-http-rule \
    --address=my-resume-ip \
    --target-http-proxy=my-resume-http-proxy \
    --ports=80 \
    --global \
    --project=my-resume-123456
```

Forwarding Rules are the **entry point** — they listen on specific
ports of the static IP and route traffic to the correct proxy.

---

## Key Concepts Learned

### Load Balancer Is Not One Thing
The "Load Balancer" is a collection of components each with one job:

```
Forwarding Rule  → accept traffic on specific port/IP
HTTPS Proxy      → terminate SSL, delegate routing
URL Map          → decide which backend receives request
Backend Bucket   → connect LB to GCS bucket
```

### Why CDN Comes After URL Map
CDN cannot be checked before the URL Map because:

```
CDN needs:    backend identity (to check correct cache)
URL Map gives: backend identity
SSL gives:    decrypted URL (needed by URL Map)

Therefore order must be:
SSL → URL Map → CDN check → GCS (if cache miss)
```

### SSL Certificate Chain of Trust
Browsers verify certificates via a chain:

```
GTS Root R1 (pre-installed in browser)
    ↓ signed
GTS Intermediate CA
    ↓ signed
example-resume-cert
    "I am example.com"
```

Browser uses Google Trust Services **public key** (pre-installed)
to verify the cryptographic signature. Only Google can create valid
signatures because only Google has the **private key**.

### Static IP Is Critical for DNS
The static IP is reserved so it never changes:

```
Dynamic IP: changes → DNS breaks ❌
Static IP:  fixed   → DNS always works ✅
```

In Step 9 we add one DNS A record:
```
example.com → 34.x.x.x (our static IP)
```

### GCS Bucket vs Backend Bucket

| | GCS Bucket | Backend Bucket |
|---|---|---|
| Contains | Actual files | Just a pointer |
| Service | Cloud Storage | Compute/LB |
| Runs code | No | No |
| Cost | Storage + egress | Part of LB cost |
| Survives LB deletion | Yes | Should be deleted |

---

## How to Remove the Load Balancer Later

When replacing with Cloudflare, delete in this exact order:

```bash
# 1. Forwarding rules first
gcloud compute forwarding-rules delete my-resume-https-rule --global
gcloud compute forwarding-rules delete my-resume-http-rule --global

# 2. Proxies
gcloud compute target-https-proxies delete my-resume-https-proxy --global
gcloud compute target-http-proxies delete my-resume-http-proxy --global

# 3. URL Maps
gcloud compute url-maps delete my-resume-urlmap --global
gcloud compute url-maps delete my-resume-http-redirect --global

# 4. SSL Certificate
gcloud compute ssl-certificates delete my-resume-cert --global

# 5. Backend Bucket
gcloud compute backend-buckets delete my-resume-backend

# 6. Static IP last
gcloud compute addresses delete my-resume-ip --global
```

> ⚠️ Always delete forwarding rules first — GCP won't let you delete
> a resource still referenced by another resource.

---

## Cost Breakdown

| Component | Cost |
|---|---|
| Global Load Balancer | ~$18/month minimum |
| PREMIUM network egress | ~$0.12/GB |
| Cloud CDN cache miss | ~$0.08/GB |
| SSL Certificate | Free |
| Static IP (in use) | Free |
| **Total estimate** | **~$18-20/month** |

The Load Balancer minimum fee dominates the cost. For a low traffic
resume site actual egress and CDN costs are negligible.

After learning, replace with **Cloudflare** for ~$0/month with
equivalent CDN performance and automatic HTTPS.

---

## Up Next

With the Load Balancer fully configured, we need to point `example.com`
to our static IP. That is covered in Part 9 — DNS Configuration.

---

*This post is part of a series on building a Hugo resume site deployed
to Google Cloud Platform with a full CI/CD pipeline via GitHub Actions.*
