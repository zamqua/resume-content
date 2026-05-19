---
title: "DNS Configuration and SSL Certificate Activation for example.com"
author: " "
date: 2026-05-05
draft: false
tags: ["hugo", "gcp", "dns", "ssl", "devops", "cloud"]
categories: ["Tutorial"]
description: "Step by step guide to configuring DNS records, activating Google-managed SSL certificates, and deploying a Hugo site to production at example.com."
showToc: true
---

This is Part 9 of a series on building and deploying a personal resume
site using Hugo and Google Cloud Platform. In this post we cover DNS
configuration, SSL certificate activation, and deploying the Hugo site
to production at `https://example.com`.

---

## Prerequisites

Before starting this step you should have:

- Load Balancer fully configured (Step 8)
- Static IP address reserved (`12.34.567.89`)
- Access to your domain DNS settings (Squarespace)
- Hugo site built and ready to deploy

---

## Understanding DNS

DNS (Domain Name System) is the internet's phone book — it maps
human readable domain names to IP addresses:

```
Phone book:   "John Smith"  → 555-1234
DNS:          "example.com"  → 12.34.567.89
```

### The DNS Hierarchy

```
Root DNS Servers (13 worldwide)
    "I know who manages .com domains"
        ↓
TLD Servers (.com)
    "I know who manages example.com"
        ↓
Authoritative DNS (Squarespace)
    "example.com = 12.34.567.89"    ← your A record
        ↓
Resolver (ISP cache)
    Caches result for TTL duration
        ↓
Your Browser
    Connects to 12.34.567.89
```

### DNS Record Types

| Type | Purpose | Example |
|---|---|---|
| **A** | Domain → IPv4 | `example.com → 12.34.567.89` |
| **AAAA** | Domain → IPv6 | `example.com → 2001:db8::1` |
| **CNAME** | Domain → domain | `www → example.com` |
| **MX** | Mail server | `example.com → mail.google.com` |
| **TXT** | Text/verification | SPF, DKIM, domain verification |
| **NS** | Nameserver | Who manages this domain |
| **CAA** | CA authorization | Which CAs can issue SSL certs |

### TXT Records — The Swiss Army Knife

TXT records store arbitrary text and are used for:

```
Domain ownership verification:
    example.com  TXT  "google-site-verification=abc123..."

SPF (email sender authorization):
    example.com  TXT  "v=spf1 include:_spf.google.com ~all"

DKIM (email content signing):
    google._domainkey.example.com  TXT  "v=DKIM1; k=rsa; p=..."

DMARC (email authentication policy):
    _dmarc.example.com  TXT  "v=DMARC1; p=reject; ..."
```

### Multiple A Records

A domain can have multiple A records for load balancing or
high availability:

```
example.com    A    34.120.123.1
example.com    A    34.120.123.2
example.com    A    34.120.123.3
```

Our setup uses ONE A record because GCP's Global Load Balancer
uses **Anycast** — one IP routes to the nearest GCP edge node
globally, providing the benefits of multiple A records without
the complexity.

---

## Add A Record in Squarespace DNS

Since the domain was purchased through Google Workspace (now
managed by Squarespace):

1. Go to Google Workspace Admin Console:
```
https://admin.google.com
```

2. Click **Domains** → **Manage Domains** → **Manage Domain**
   (redirects to Squarespace Domains)

3. Click **DNS** → **DNS Settings** → **Custom Records**

4. Add A record:
```
Type:  A
Host:  @          ← @ means root domain example.com
Value: 12.34.567.89  ← Load Balancer static IP
TTL:   3600           ← 1 hour cache
```

5. Optionally add www CNAME:
```
Type:  CNAME
Host:  www
Value: example.com.    ← trailing dot required
TTL:   3600
```

>[!NOTE] CNAME creates an alias (www.zamqua.com) for zamqua.com.

>[!NOTE] The trailing period after the url is to make it an absolute, fully qualified path (DNS treats it as complete, final path).

---

## Verify DNS Propagation

DNS changes propagate based on TTL — can take 5 minutes to
48 hours. Verify with `dig`:

```bash
dig example.com A
```

Expected output:
```
;; ANSWER SECTION:
example.com.    3600    IN    A    12.34.567.89
```

Check propagation worldwide:
```
https://dnschecker.org/#A/example.com
```

---

## SSL Certificate Activation

### How Google-Managed Certificates Work

Google verifies domain ownership before signing the certificate:

```
Stage 1: PROVISIONING
    Google checks DNS A record points to LB ✅
    Google checks LB responds on port 80/443 ✅
    Google verifies domain ownership ✅

Stage 2: PROVISIONING
    "Working with CA to sign certificate"
    Google Trust Services signs the cert
    Takes 10-30 minutes

Stage 3: ACTIVE
    Certificate live ✅
    https://example.com works ✅
```

### Monitor Certificate Status

```bash
gcloud compute ssl-certificates describe my-resume-cert \
    --global --project=$PJ \
    --format="get(managed.status)"
```

Status values:
```
PROVISIONING  → verification/signing in progress
ACTIVE        → certificate live ✅
FAILED_NOT_VISIBLE → LB not responding correctly
```

### Troubleshooting FAILED_NOT_VISIBLE

This error means Google's provisioning bot cannot reach your LB.
Check each component:

```bash
# Verify DNS
dig example.com A

# Verify LB IP matches DNS
gcloud compute addresses describe my-resume-ip \
    --global --project=$PJ --format="get(address)"

# Verify forwarding rules exist
gcloud compute forwarding-rules list --global --project=$PJ

# Test HTTP response
curl -Iv http://example.com
```

Expected HTTP response:
```
HTTP/1.1 301 Moved Permanently
Location: https://example.com/
```

If HTTP returns 301 correctly but cert still shows
`FAILED_NOT_VISIBLE`, delete and recreate the certificate:

```bash
# Detach cert
gcloud compute target-https-proxies update my-resume-https-proxy \
    --clear-ssl-certificates \
    --global --project=$PJ

# Delete
gcloud compute ssl-certificates delete my-resume-cert \
    --global --project=$PJ

# Recreate
gcloud compute ssl-certificates create my-resume-cert \
    --domains=example.com \
    --global --project=$PJ

# Reattach
gcloud compute target-https-proxies update my-resume-https-proxy \
    --ssl-certificates=my-resume-cert \
    --global --project=$PJ
```

---

## Fix HTTP Redirect Port Issue

By default GCP's HTTP → HTTPS redirect shows port 443 explicitly:

```
❌ Location: https://example.com:443/
✅ Location: https://example.com/
```

Fix by specifying `hostRedirect` in the redirect URL map:

```bash
cat > /tmp/http-redirect.yaml << 'EOF'
name: my-resume-http-redirect
defaultUrlRedirect:
  redirectResponseCode: MOVED_PERMANENTLY_DEFAULT
  httpsRedirect: true
  hostRedirect: "example.com"
EOF

gcloud compute url-maps import my-resume-http-redirect \
    --source=/tmp/http-redirect.yaml \
    --global --project=$PJ
```

---

## Deploy Hugo Site to Production

### Update baseURL

Once SSL is active update `hugo.toml`:

```toml
baseURL = 'https://example.com/'
```

Remove temporary settings if present:
```toml
# Remove these if present
uglyURLs = true
```

### Always Clean Build Before Deploying

Stale files in `public/` can cause issues — links broken,
wrong URLs, outdated assets:

```bash
# Remove stale build
rm -rf public/

# Fresh build
hugo

# Deploy to GCS
hugo deploy
```

> ⚠️ Always run `rm -rf public/` before `hugo` when changing
> `baseURL` or theme settings. Hugo may not rebuild all files
> if it thinks they haven't changed.

### Verify Built Files

Check generated HTML has correct URLs:

```bash
grep -i "href\|src" public/index.html | head -20
```

Should show:
```html
href="https://example.com/experience/"
href="https://example.com/css/main.css"
```

Not:
```html
href="https://storage.googleapis.com/example.com/experience/"
href="/experience/"
```

---

## Re-authentication Issues

Hugo deploy uses **Application Default Credentials (ADC)** —
separate from regular gcloud credentials:

```
gcloud auth login
    → credentials for gcloud CLI commands
    → NOT used by hugo deploy

gcloud auth application-default login
    → credentials for applications and SDKs
    → USED by hugo deploy ✅
```

If `hugo deploy` fails with `invalid_grant` or `reauth` error:

```bash
# Refresh CLI credentials
gcloud auth login

# Refresh application credentials
gcloud auth application-default login

# Retry deployment
hugo deploy
```

> 💡 This manual re-authentication is why CI/CD with Service
> Accounts is important — Service Account credentials never
> expire and don't require human interaction.

---

## Final Verification

Test all URLs in browser:

```
https://example.com              ← homepage 🔒 ✅
http://example.com               ← redirects to https ✅
https://example.com/experience/  ← experience page ✅
https://example.com/education/   ← education page ✅
https://example.com/skills/      ← skills page ✅
https://example.com/projects/    ← projects page ✅
```

---

## Key Concepts Learned

### DNS
- DNS maps domain names to IP addresses via distributed hierarchy
- A records point domains to IPv4 addresses
- TTL controls how long DNS resolvers cache records
- Multiple A records enable load balancing via DNS Round Robin
- Anycast (our setup) achieves same result with single IP

### SSL Certificate Lifecycle
- Google verifies domain ownership via DNS before signing
- Provisioning takes 10-30 minutes after DNS propagates
- `FAILED_NOT_VISIBLE` means LB not responding to Google's bot
- Recreating certificate resets the verification process

### Application Default Credentials
- Two separate credential systems in gcloud
- `gcloud auth login` → for CLI commands
- `gcloud auth application-default login` → for applications
- Both expire — CI/CD uses Service Accounts instead

### Stale Build Directory
- Always `rm -rf public/` before rebuilding after config changes
- Hugo may skip rebuilding unchanged files
- Stale files cause broken links and wrong asset URLs

---

## Current Infrastructure State

```
✅ DNS:         example.com → 12.34.567.89
✅ SSL:         ACTIVE (Google-managed, auto-renews)
✅ HTTP:        301 redirect to HTTPS
✅ HTTPS:       example.com serving Hugo resume
✅ CDN:         Cloud CDN caching at GCP edge nodes
✅ Site:        All pages and links working correctly
```

---

## Up Next

With the site live at `https://example.com`, we need to automate
deployments so every `git push` automatically builds and deploys
the site. That requires setting up a **Service Account and IAM
permissions** covered in Part 10, followed by the **GitHub Actions
CI/CD pipeline** in Part 11.

---

*This post is part of a series on building a Hugo resume site deployed
to Google Cloud Platform with a full CI/CD pipeline via GitHub Actions.*
