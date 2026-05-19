---
title: "Deploying Hugo Resume Site with Cloudflare and GCS — Zero Cost Hosting"
date: 2026-05-12
draft: false
tags: ["hugo", "cloudflare", "gcs", "devops", "cicd", "dns"]
categories: ["Tutorial"]
description: "Step by step guide to hosting a Hugo static site on Google Cloud Storage with Cloudflare for free SSL, CDN, and DNS — at ~$0/month."
showToc: true
---

This post covers deploying a Hugo resume site using **Cloudflare + Google
Cloud Storage** — a zero cost alternative to the GCP Load Balancer setup.
The site is served at `https://mazam.me` with free SSL, CDN, and DNS
via Cloudflare.

---

## Architecture

```
Visitor
    ↓
mazam.me (Cloudflare DNS)
    ↓
Cloudflare (free SSL + CDN + DDoS protection)
    ↓
gs://mazam.me (Google Cloud Storage)
    ↓
Hugo static site files
```

---

## Cost Comparison

| Component | GCP Load Balancer | Cloudflare |
|---|---|---|
| SSL Certificate | Free (Google-managed) | Free (Cloudflare) |
| CDN | ~$0.08/GB | Free |
| Load Balancer | ~$18/month | Not needed |
| DNS | Google Cloud DNS | Free (Cloudflare) |
| DDoS Protection | Basic | Advanced (free) |
| **Total** | **~$18/month** | **~$0.01/month** |

---

## Prerequisites

- Hugo site ready and working locally
- GitHub repository with Hugo source
- GCP project with `gcloud` CLI configured
- Cloudflare account (free at cloudflare.com)
- Domain registered (mazam.me via Squarespace)

---

## Step 1 — Register Domain

`mazam.me` was purchased through Squarespace. After purchase
DNS propagation takes 1-4 hours.

Verify propagation:
```bash
dig mazam.me A +short
```

Squarespace default parking IPs appear when ready:
```
198.185.159.145
198.49.23.145
198.49.23.144
198.185.159.144
```

---

## Step 2 — Set Up Cloudflare

### Create Cloudflare Account
```
https://cloudflare.com
```
Sign up for free account.

### Add Domain to Cloudflare
1. Click **"Add a Site"**
2. Enter `mazam.me`
3. Select **Free plan** ($0/month)
4. Cloudflare scans and imports existing DNS records

### Note Your Cloudflare Nameservers
Cloudflare assigns two nameservers:
```
e.g.:
    aria.ns.cloudflare.com
    leo.ns.cloudflare.com
```

---

## Step 3 — Disable DNSSEC Before Changing Nameservers

DNSSEC must be disabled before changing nameservers to prevent
DNS resolution failures during transition:

```
Squarespace Domains → mazam.me
    → Security → DNSSEC
    → Disable DNSSEC → Confirm
```

Why:
```
DNSSEC signs records with Squarespace keys
Changing to Cloudflare invalidates those signatures
Browsers reject all DNS responses ❌

Solution:
    Disable DNSSEC → Change nameservers → Cloudflare re-enables ✅
```

---

## Step 4 — Update Nameservers at Squarespace

```
Squarespace Domains → mazam.me
    → Nameservers
    → Use custom nameservers
    → Enter Cloudflare nameservers
    → Save
```

Verify propagation:
```bash
dig mazam.me NS +short
```

Should show Cloudflare nameservers (may take 30-60 minutes).

---

## Step 5 — Configure Cloudflare DNS Records

### Delete Squarespace Parking A Records
Remove all four default Squarespace IPs:
```
198.185.159.145
198.49.23.145
198.49.23.144
198.185.159.144
```

### Add GCS CNAME Record
```
Type:    CNAME
Name:    @
Target:  c.storage.googleapis.com
TTL:     Auto
Proxy:   ✅ Proxied (orange cloud ON)
```

### Add www CNAME
```
Type:    CNAME
Name:    www
Target:  mazam.me
TTL:     Auto
Proxy:   ✅ Proxied (orange cloud ON)
```

> 💡 The orange cloud (Proxied) is important — it enables
> Cloudflare CDN, SSL, and DDoS protection. Without it,
> traffic bypasses Cloudflare entirely.

### Why CNAME Not A Record for Root Domain?
```
GCS uses:  c.storage.googleapis.com (no fixed IP)
A record:  requires fixed IP → can't use for GCS ❌
CNAME:     points to hostname → works with GCS ✅

Cloudflare uses "CNAME Flattening" to allow CNAME on
root domain (@) — a standard DNS restriction that
Cloudflare solves automatically.
```

---

## Step 6 — Configure Cloudflare SSL

```
Cloudflare → mazam.me → SSL/TLS → Overview
    → Encryption mode: Full (strict)
```

Enable additional security:
```
SSL/TLS → Edge Certificates
    → Always Use HTTPS:          ON ✅
    → Automatic HTTPS Rewrites:  ON ✅
```

SSL encryption modes explained:
```
Off           → HTTP only ❌
Flexible      → HTTPS to Cloudflare, HTTP to GCS ⚠️
Full          → HTTPS everywhere, self-signed cert ok
Full (strict) → HTTPS with valid certificate ✅ ← use this
```

---

## Step 7 — Create GCS Bucket

### Verify Domain Ownership First

GCP requires domain ownership verification before creating
a bucket named after a domain.

```
https://search.google.com/search-console
```

Add `mazam.me` as a property and verify via DNS TXT record:

```
Cloudflare DNS → Add TXT record:
    Type:  TXT
    Name:  @
    Value: google-site-verification=xxxxx...
```

> ⚠️ Two verifications may be needed if different Google
> accounts own the domain vs the GCP project. Add both
> TXT records — they don't interfere with each other.

### Create Bucket

```bash
gcloud storage buckets create gs://mazam.me \
    --project=$PJ \
    --location=us-central1 \
    --uniform-bucket-level-access
```

### Configure Public Access

```bash
# Disable public access prevention
gcloud storage buckets update gs://mazam.me \
    --no-public-access-prevention

# Make bucket publicly readable
gcloud storage buckets add-iam-policy-binding gs://mazam.me \
    --member="allUsers" \
    --role="roles/storage.objectViewer"

# Configure static website hosting
gcloud storage buckets update gs://mazam.me \
    --web-main-page-suffix=index.html \
    --web-error-page=404.html
```

### Grant GitHub Actions Access

```bash
gcloud storage buckets add-iam-policy-binding gs://mazam.me \
    --member="serviceAccount:github-actions@my-resume-123456.iam.gserviceaccount.com" \
    --role="roles/storage.objectAdmin"
```

---

## Step 8 — Update Workload Identity for New Repo

The existing Workload Identity Provider only allowed the
original repository. Update to allow the new repo too:

```bash
gcloud iam workload-identity-pools providers update-oidc github-provider \
    --location="global" \
    --workload-identity-pool=github-pool \
    --attribute-condition="assertion.repository=='zamqua/my-resume' || assertion.repository=='zamqua/my-resume-cf'" \
    --project=$PJ
```

Add service account binding for new repo:

```bash
export PROJECT_NUMBER=$(gcloud projects describe $PJ \
    --format="get(projectNumber)")

gcloud iam service-accounts add-iam-policy-binding \
    github-actions@my-resume-123456.iam.gserviceaccount.com \
    --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/zamqua/my-resume-cf" \
    --role="roles/iam.workloadIdentityUser" \
    --project=$PJ
```

---

## Step 9 — Create New GitHub Repository

```
https://github.com/new
    Name:        my-resume-cf
    Description: Hugo resume site — Cloudflare + GCS deployment
    Visibility:  Public
    Initialize:  NO
```

---

## Step 10 — Create Cloudflare API Token

For GitHub Actions to purge Cloudflare cache after deployment:

```
Cloudflare → My Profile → API Tokens
    → Create Token
    → Create Custom Token

Token name:   GitHub Actions Cache Purge
Permissions:  Zone → Cache Purge → Purge
Zone:         Include → Specific Zone → mazam.me

→ Create Token → Copy immediately
```

Get Zone ID:
```
Cloudflare → mazam.me → Overview → Zone ID (right panel)
```

---

## Step 11 — Add GitHub Secrets

```
https://github.com/zamqua/my-resume-cf/settings/secrets/actions
```

Add five secrets:

```
GCP_PROJECT_ID
    → my-resume-123456

GCP_SERVICE_ACCOUNT
    → github-actions@my-resume-123456.iam.gserviceaccount.com

GCP_WORKLOAD_IDENTITY_PROVIDER
    → projects/12345678901/locations/global/
      workloadIdentityPools/github-pool/providers/github-provider

CF_ZONE_ID
    → from Cloudflare dashboard → mazam.me → Overview

CF_API_TOKEN
    → from Cloudflare API token created above
```

---

## Step 12 — Update Hugo Configuration

```toml
# hugo.toml
baseURL = 'https://mazam.me/'

[deployment]
  [[deployment.targets]]
    name = "gcs"
    URL  = "gs://mazam.me"

  [[deployment.matchers]]
    pattern = "^.+\\.html$"
    cacheControl = "max-age=60, no-transform, public"
    gzip = true

  [[deployment.matchers]]
    pattern = "^.+\\.(js|css|svg|ttf|woff|woff2)$"
    cacheControl = "max-age=31536000, no-transform, public"
    gzip = true

  [[deployment.matchers]]
    pattern = "^.+\\.(png|jpg|gif|webp|ico)$"
    cacheControl = "max-age=2592000, no-transform, public"
    gzip = false
```

---

## Step 13 — GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy Hugo Site to GCS via Cloudflare

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
  HUGO_VERSION: 0.160.1

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    name: Build and Deploy
    runs-on: ubuntu-latest

    steps:
      # Step 1 — Checkout code + PaperMod submodule
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0

      # Step 2 — Install Hugo Extended with Deploy support
      - name: Setup Hugo
        run: |
          wget -O hugo.tar.gz https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_withdeploy_${HUGO_VERSION}_linux-amd64.tar.gz
          tar -xzf hugo.tar.gz
          sudo mv hugo /usr/local/bin/hugo
          hugo version

      # Step 3 — Build site
      - name: Build site
        run: |
          rm -rf public/
          hugo --minify
        env:
          HUGO_ENVIRONMENT: production

      # Step 4 — Authenticate to GCP
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      # Step 5 — Setup gcloud CLI
      - name: Setup gcloud CLI
        uses: google-github-actions/setup-gcloud@v2
        with:
          project_id: ${{ secrets.GCP_PROJECT_ID }}

      # Step 6 — Deploy to GCS
      - name: Deploy to GCS
        run: |
          hugo deploy \
            --target=gcs \
            --maxDeletes=-1

      # Step 7 — Purge Cloudflare Cache
      - name: Purge Cloudflare Cache
        run: |
          curl -X POST \
            "https://api.cloudflare.com/client/v4/zones/${{ secrets.CF_ZONE_ID }}/purge_cache" \
            -H "Authorization: Bearer ${{ secrets.CF_API_TOKEN }}" \
            -H "Content-Type: application/json" \
            --data '{"purge_everything":true}'

      # Step 8 — Confirm
      - name: Deployment complete
        run: |
          echo "✅ Site deployed to gs://mazam.me"
          echo "🌐 https://mazam.me"
```

### Key Difference From GCP LB Workflow

```
GCP LB workflow:          Cloudflare workflow:
hugo deploy               hugo deploy
  --invalidateCDN    →      (removed — no Cloud CDN)
                          + Purge Cloudflare Cache step
```

---

## Step 14 — Deploy

```bash
cd ~/Projects/my-resume-cf

# Clean build
rm -rf public/
hugo

# Deploy manually first time
hugo deploy

# Then push to trigger CI/CD
git add .
git commit -m "Initial commit — Hugo resume with Cloudflare"
git push -u origin main
```

---

## Step 15 — Verify

```bash
# Check DNS
dig mazam.me NS +short       # Cloudflare nameservers
dig mazam.me CNAME +short    # c.storage.googleapis.com

# Check site
curl -I https://mazam.me
```

SSL issuer should show Cloudflare:
```
issuer: CN=Cloudflare Inc ECC CA-3
```

---

## Key Concepts Learned

### CNAME Flattening
Standard DNS forbids CNAME on root domain (@). Cloudflare
solves this with CNAME Flattening — internally resolves
the CNAME and returns an A record to browsers.

### DNSSEC Transfer Process
```
1. Disable DNSSEC (old provider)
2. Change nameservers
3. Re-enable DNSSEC (new provider)
Never change nameservers with DNSSEC enabled ❌
```

### GCS Domain Ownership Verification
GCP requires Search Console verification before creating
a bucket named after a domain. Multiple Google accounts
can verify the same domain — each adds their own TXT record.

### Cloudflare Cache Purge vs Cloud CDN Invalidation
```
Cloud CDN:   hugo deploy --invalidateCDN
Cloudflare:  curl to /purge_cache API endpoint
Both:        ensure visitors see new content immediately
```

### Orange Cloud = Proxied
```
Orange cloud ON  → traffic through Cloudflare ✅
                   SSL, CDN, DDoS protection active
Grey cloud OFF   → traffic bypasses Cloudflare ❌
                   direct to origin, no protection
```

---

## Troubleshooting

### 403 on Bucket Creation
```
Error: Another user owns the domain mazam.me
Fix:   Verify domain in Google Search Console
       under the GCP account (mazam@zamqua.com)
       Add TXT record in Cloudflare DNS
```

### Permission Denied on Deploy
```
Error: does not have storage.objects.create access
Fix:   gcloud storage buckets add-iam-policy-binding gs://mazam.me
         --member="serviceAccount:github-actions@..."
         --role="roles/storage.objectAdmin"
```

### Workload Identity Rejected
```
Error: The given credential is rejected by attribute condition
Fix:   Update attribute condition to include new repo:
       assertion.repository=='zamqua/my-resume-cf'
```

### Site Shows GCS XML Instead of HTML
```
Error: Bucket listing shown instead of website
Fix:   Verify Cloudflare proxy is ON (orange cloud)
       Verify CNAME points to c.storage.googleapis.com
       Verify website hosting configured on bucket
```

---

## Final Architecture

```
Developer:
    git push origin main
            ↓
GitHub Actions (zamqua/my-resume-cf):
    Build Hugo → Deploy to gs://mazam.me → Purge CF cache
            ↓
gs://mazam.me (Google Cloud Storage):
    Hugo static files
            ↓
Cloudflare (free):
    DNS + SSL + CDN + DDoS protection
            ↓
https://mazam.me ✅
```

---

## Monthly Cost

```
GCS Storage:     ~$0.01/month
Cloudflare:      $0/month (free plan)
GitHub Actions:  $0/month (free tier)
──────────────────────────────────────
Total:           ~$0.01/month 🎉
```

---

*Part of a series on building and deploying a Hugo resume site.
Source code available at
[github.com/zamqua/my-resume-cf](https://github.com/zamqua/my-resume-cf).*
