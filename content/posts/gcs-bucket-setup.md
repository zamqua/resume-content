---
title: "Creating and Configuring a GCS Bucket for Static Website Hosting"
author: " "
date: 2026-05-04
draft: false
tags: ["hugo", "gcp", "gcs", "devops", "cloud", "storage"]
categories: ["Tutorial"]
description: "Step by step guide to creating a Google Cloud Storage bucket for hosting a Hugo static site, including public access configuration, org policies, and static website hosting setup."
showToc: true
---

This is Part 7 of a series on building and deploying a personal resume
site using Hugo and Google Cloud Platform. In this post we cover creating
and configuring a GCS bucket for static website hosting — including
navigating Google Workspace org policies that block public access by default.

---

## Prerequisites

Before starting this step you should have:

- GCP project configured (`my-resume-123456`)
- Required APIs enabled
- `gcloud` CLI authenticated
- Hugo site built and ready to deploy

---

## Bucket Naming Rule

For custom domain hosting, GCS requires the bucket name to **exactly
match your domain name** when using direct GCS static hosting:

```
Domain:       example.com
Bucket name:  example.com
```

When behind a Load Balancer this is not strictly required — but it is
a good convention for clarity and future flexibility.

---

## Create the Bucket

Use `us-central1` single region instead of multi-region for cost savings:

```bash
gcloud storage buckets create gs://example.com \
    --project=my-resume-123456 \
    --location=us-central1 \
    --uniform-bucket-level-access
```

Key flags:

| Flag | Value | Reason |
|---|---|---|
| `--location` | `us-central1` | Cheapest GCP region |
| `--uniform-bucket-level-access` | enabled | Consistent permissions |

### Cost Comparison

| Option | Storage cost | Notes |
|---|---|---|
| Multi-region US | $0.026/GB | Higher availability |
| `us-central1` | $0.020/GB | 23% cheaper, sufficient for resume |

---

## Navigating Google Workspace Org Policies

If your GCP account is under a Google Workspace organization you will
encounter two org policies that block public bucket access by default.
These must be addressed before the bucket can be made public.

### Policy 1 — Public Access Prevention

This policy blocks `allUsers` from being added to bucket IAM policies:

```
constraints/storage.publicAccessPrevention = enforced
```

Check the effective policy:

```bash
gcloud org-policies describe storage.publicAccessPrevention \
    --effective \
    --organization=YOUR_ORG_ID
```

Override at org level:

```bash
cat > /tmp/public-access-policy.yaml << 'EOF'
name: organizations/YOUR_ORG_ID/policies/storage.publicAccessPrevention
spec:
  rules:
  - enforce: false
EOF

gcloud org-policies set-policy /tmp/public-access-policy.yaml
```

### Policy 2 — Domain Restricted Sharing

This policy restricts IAM bindings to principals within your organization
domain only — blocking `allUsers`:

```
constraints/iam.allowedPolicyMemberDomains
    = is:principalSet://iam.googleapis.com/organizations/YOUR_ORG_ID
```

Check the effective policy:

```bash
gcloud org-policies describe iam.allowedPolicyMemberDomains \
    --effective \
    --organization=YOUR_ORG_ID
```

Add exception for tagged public-facing resources:

```bash
cat > /tmp/domain-policy.yaml << 'EOF'
name: organizations/YOUR_ORG_ID/policies/iam.allowedPolicyMemberDomains
spec:
  rules:
  - condition:
      expression: "resource.matchTag('PROJECT_ID/public-facing', 'true')"
      title: Allow allUsers for public facing resources
    allowAll: true
  - values:
      allowedValues:
      - "is:principalSet://iam.googleapis.com/organizations/YOUR_ORG_ID"
EOF

gcloud org-policies set-policy /tmp/domain-policy.yaml
```

> ⚠️ Org policy changes can take **2-10 minutes** to propagate. If
> commands fail immediately after setting a policy, wait and retry.

---

## Tag-Based Security (Enterprise Pattern)

Rather than disabling org policies globally, use **resource tags** to
apply exceptions only to resources that need them. This is the
recommended enterprise pattern.

### Create Tag Key and Value

```bash
# Create tag key
gcloud resource-manager tags keys create public-facing \
    --parent=projects/my-resume-123456 \
    --description="Marks resources that are intentionally public facing"

# Get tag key ID
gcloud resource-manager tags keys list \
    --parent=projects/my-resume-123456

# Create tag value using numeric key ID
gcloud resource-manager tags values create true \
    --parent=tagKeys/NUMERIC_KEY_ID \
    --description="Resource is intentionally public facing"
```

### Attach Tag to Bucket

```bash
gcloud resource-manager tags bindings create \
    --tag-value=tagValues/NUMERIC_VALUE_ID \
    --parent=//storage.googleapis.com/projects/_/buckets/example.com \
    --location=us-central1
```

### Verify Tag Is Attached

```bash
gcloud resource-manager tags bindings list \
    --parent=//storage.googleapis.com/projects/_/buckets/example.com \
    --location=us-central1
```

### The Security Model

```
Org policy: public access = ENFORCED
    EXCEPT resources tagged public-facing=true

Result:
    gs://example.com (tagged)      → can be public ✅
    gs://any-other-bucket         → stays private ✅
```

---

## Make Bucket Public

Once org policies allow it:

```bash
gcloud storage buckets add-iam-policy-binding gs://example.com \
    --member="allUsers" \
    --role="roles/storage.objectViewer"
```

Verify the binding was added:
```yaml
- members:
  - allUsers
  role: roles/storage.objectViewer    ← ✅ public access granted
```

---

## Configure Static Website Hosting

```bash
gcloud storage buckets update gs://example.com \
    --web-main-page-suffix=index.html \
    --web-error-page=404.html
```

This tells GCS:
- Serve `index.html` when a directory is requested
- Serve `404.html` for missing pages

Verify via console at:
```
https://console.cloud.google.com/storage/browser/example.com
```

---

## Configure Hugo Deployment

Add deployment configuration to `hugo.toml`:

```toml
[deployment]
  [[deployment.targets]]
    name = "gcs"
    URL  = "gs://example.com"

  [[deployment.matchers]]
    # Cache HTML for 1 minute
    pattern = "^.+\\.html$"
    cacheControl = "max-age=60, no-transform, public"
    gzip = true

  [[deployment.matchers]]
    # Cache assets for 1 year
    pattern = "^.+\\.(js|css|svg|ttf|woff|woff2)$"
    cacheControl = "max-age=31536000, no-transform, public"
    gzip = true

  [[deployment.matchers]]
    # Cache images for 1 month
    pattern = "^.+\\.(png|jpg|gif|webp|ico)$"
    cacheControl = "max-age=2592000, no-transform, public"
    gzip = false
```

---

## Deploy Hugo Site to GCS

```bash
cd ~/Projects/my-resume

# Build site
hugo

# Deploy to GCS bucket
hugo deploy
```

Hugo reads the `[deployment]` section from `hugo.toml` and uploads
the `public/` directory to `gs://example.com` automatically using
your `gcloud` credentials.

---

## GCS URL Limitations

GCS static hosting has two different URL endpoints with different behaviors:

| URL | Behavior |
|---|---|
| `https://storage.googleapis.com/example.com/` | Storage API — no directory routing |
| `http://example.com.storage.googleapis.com` | Website endpoint — supports directory routing |

### The Directory Routing Problem

Hugo generates pretty URLs like `/education/` but GCS looks for exact
file keys. Without proper routing `/education/` returns a `NoSuchKey`
error because the actual file is `education/index.html`.

This is solved by either:

1. **Load Balancer** (Step 8) — handles routing automatically ✅
2. **Cloudflare Worker** — rewrites URLs before hitting GCS ✅
3. **Hugo `uglyURLs`** — generates `.html` links instead of directories

```toml
# Temporary workaround for direct GCS testing
uglyURLs = true    # generates /education.html instead of /education/
```

> 💡 Always use pretty URLs (`uglyURLs = false`) in production — they
> look more professional and are better for SEO.

---

## Key Lessons Learned

### GCP Concepts
- **Bucket naming** — matches domain for direct hosting, optional behind LB
- **Uniform bucket level access** — consistent permissions across all objects
- **Org policies** — override project permissions, must be addressed at org level
- **Policy propagation** — changes take 2-10 minutes to take effect
- **Tag-based policies** — enterprise pattern for selective policy exceptions

### Two Blocking Org Policies for Public Buckets
1. `storage.publicAccessPrevention` — blocks allUsers on buckets
2. `iam.allowedPolicyMemberDomains` — restricts IAM to org members only

### Cache Strategy
- HTML files: short cache (1 min) — content changes frequently
- CSS/JS/fonts: long cache (1 year) — content rarely changes
- Images: medium cache (1 month) — balance between freshness and performance

---

## Current Infrastructure State

```
✅ GCS Bucket:    gs://example.com (us-central1)
✅ Access:        allUsers = objectViewer (public)
✅ Website:       index.html / 404.html configured
✅ Hugo deploy:   hugo.toml deployment section configured
✅ Site files:    uploaded to bucket
✅ Tag:           public-facing=true attached to bucket
```

---

## Up Next

With the GCS bucket configured and site files uploaded, we are ready
to set up the **Cloud Load Balancer** to serve the site at
`https://example.com` with proper HTTPS and directory routing.
That is covered in Part 8 of this series.

---

*This post is part of a series on building a Hugo resume site deployed
to Google Cloud Platform with a full CI/CD pipeline via GitHub Actions.*
