---
title: "GitHub Actions CI/CD Pipeline for Hugo Site Deployment"
author: " "
date: 2026-05-05
draft: false
tags: ["hugo", "gcp", "cicd", "github-actions", "devops", "automation"]
categories: ["Tutorial"]
description: "Step by step guide to creating a GitHub Actions CI/CD pipeline that automatically builds and deploys a Hugo site to Google Cloud Storage on every git push."
showToc: true
---

This is Part 11 of a series on building and deploying a personal resume
site using Hugo and Google Cloud Platform. In this post we cover creating
a GitHub Actions CI/CD pipeline that automatically builds and deploys
the Hugo site to GCS on every push to the main branch.

---

## What is CI/CD?

CI/CD stands for Continuous Integration / Continuous Deployment:

```
Continuous Integration (CI):
    Every push triggers automated build and tests
    "Does the code build correctly?"

Continuous Deployment (CD):
    Every successful build deploys automatically
    "Get working code to production immediately"
```

For our Hugo site:
```
git push origin main
        ↓
GitHub Actions automatically:
    1. Checks out code
    2. Installs Hugo
    3. Builds site
    4. Authenticates to GCP
    5. Deploys to GCS
    6. Invalidates CDN cache
        ↓
https://example.com updated ✅
```

---

## GitHub Actions Concepts

### What is GitHub Actions?

GitHub Actions is a CI/CD platform built into GitHub. It runs
automated workflows on **runners** — fresh Ubuntu VMs that GitHub
spins up for each workflow run.

```
Fresh Ubuntu VM per run:
    2 CPU cores
    7GB RAM
    14GB SSD
    Nothing pre-installed
        ↓
Runs your workflow steps
        ↓
VM destroyed when complete
```

### Why a Fresh VM Every Time?

```
✅ Reproducible — same environment every run
✅ Secure — no leftover files from previous runs
✅ Clean — no dependency conflicts
✅ Isolated — cannot affect other users
⚠️ Must install everything from scratch (Hugo, gcloud, etc.)
```

### GitHub Actions Pricing

```
Free tier:
    2,000 minutes/month
    500MB storage

Our pipeline:  ~45 seconds per run
Free deploys:  ~2,666 per month

For a resume site: effectively unlimited ✅
```

### Actions Marketplace

Reusable pre-built steps called **Actions** are available:

```yaml
# Instead of writing complex code yourself:
- uses: actions/checkout@v4

# GitHub builds the equivalent of:
git clone https://github.com/example/my-resume .
git checkout main
git submodule update --init --recursive
# + many more edge cases handled automatically
```

Actions syntax:
```yaml
uses: owner/repository@version
      ──────┬──────────  ──┬──
            │               │
            │               └── version tag
            │                   @v4 = major version
            │                   @v4.1.1 = exact version
            │                   @abc1234 = commit hash
            │
            └── GitHub repository (github.com/owner/repository)
```

Version pinning importance:
```yaml
uses: actions/checkout@v4       # safe — major version ✅
uses: actions/checkout@main     # dangerous — can break ❌
uses: actions/checkout@v4.1.1   # most stable ✅
```

---

## Hugo Installation Challenge

### The Problem

Standard Hugo installation actions don't include `+withdeploy`:

```
peaceiris/actions-hugo installs:
    hugo v0.160.1+extended           ← missing withdeploy ❌

We need:
    hugo v0.160.1+extended+withdeploy ← has hugo deploy ✅
```

### Hugo Edition Naming Convention

Hugo releases multiple editions per version:

```
hugo_0.160.1_linux-amd64.tar.gz
    → standard edition

hugo_extended_0.160.1_linux-amd64.tar.gz
    → extended edition (SCSS support)

hugo_extended_withdeploy_0.160.1_linux-amd64.tar.gz
    → extended + deploy edition ✅ (what we need)
```

> Note: Older versions used `with_deploy` (with underscore).
> Newer versions use `withdeploy` (no underscore).
> Always verify the exact filename for your version:

```bash
curl -s https://api.github.com/repos/gohugoio/hugo/releases/tags/v0.160.1 \
    | grep "browser_download_url" \
    | grep "extended" \
    | grep "linux-amd64"
```

### Solution — Install Hugo Manually

```yaml
- name: Setup Hugo
  run: |
    wget -O hugo.tar.gz https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_withdeploy_${HUGO_VERSION}_linux-amd64.tar.gz
    tar -xzf hugo.tar.gz
    sudo mv hugo /usr/local/bin/hugo
    hugo version
```

`HUGO_VERSION` is defined at workflow level so it can be updated
in one place and used throughout the workflow.

---

## GitHub Personal Access Token Scopes

To push workflow files to GitHub your PAT needs the `workflow` scope:

```
Required scopes:
    ✅ repo      — read/write code files
    ✅ workflow  — read/write .github/workflows/ files
```

Without `workflow` scope:
```
! [remote rejected] main -> main
  (refusing to allow a Personal Access Token to create
   or update workflow without `workflow` scope)
```

Add workflow scope at:
```
https://github.com/settings/tokens
```

---

## GitHub Secrets

Sensitive values are stored as GitHub Secrets — never in code:

```
https://github.com/example/my-resume/settings/secrets/actions
```

Three secrets required:

```
GCP_PROJECT_ID
    → my-resume-123456

GCP_WORKLOAD_IDENTITY_PROVIDER
    → projects/12345678901/locations/global/
      workloadIdentityPools/github-pool/providers/github-provider

GCP_SERVICE_ACCOUNT
    → github-actions@my-resume-123456.iam.gserviceaccount.com
```

Referenced in workflow as:
```yaml
${{ secrets.GCP_PROJECT_ID }}
${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
${{ secrets.GCP_SERVICE_ACCOUNT }}
```

Secrets are:
```
✅ Encrypted at rest
✅ Never shown in logs
✅ Only accessible to workflow runs
✅ Not visible to repository forks
```

---

## The Complete Workflow File

```yaml
name: Deploy Hugo Site to GCS

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
  HUGO_VERSION: 0.160.1 

on:
  push:
    branches:
      - main
  workflow_dispatch:    # allows manual trigger from GitHub UI

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
  HUGO_VERSION: 0.160.1    # pinned — change here to upgrade

permissions:
  contents: read
  id-token: write          # required for Workload Identity

jobs:
  deploy:
    name: Build and Deploy
    runs-on: ubuntu-latest

    steps:
      # Step 1 — Checkout code including PaperMod submodule
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          submodules: true      # fetches PaperMod at pinned commit
          fetch-depth: 0        # full history for Hugo lastmod dates

      # Step 2 — Install Hugo Extended with Deploy support
      - name: Setup Hugo
        run: |
          wget -O hugo.tar.gz https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_withdeploy_${HUGO_VERSION}_linux-amd64.tar.gz
          tar -xzf hugo.tar.gz
          sudo mv hugo /usr/local/bin/hugo
          hugo version

      # Step 3 — Build Hugo site
      - name: Build site
        run: |
          rm -rf public/
          hugo --minify
        env:
          HUGO_ENVIRONMENT: production

      # Step 4 — Authenticate to GCP via Workload Identity
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
            --maxDeletes=-1 \
            --invalidateCDN

      # Step 7 — Confirm deployment
      - name: Verify deployment
        run: |
          echo "✅ Site deployed successfully!"
          echo "🌐 https://example.com"
```

---

## Workflow Components Explained

### Triggers
```yaml
on:
  push:
    branches:
      - main          # auto-deploy on every push to main
  workflow_dispatch:  # manual trigger from GitHub UI
```

### Permissions
```yaml
permissions:
  contents: read      # read repository code
  id-token: write     # request JWT token from GitHub
                      # REQUIRED for Workload Identity ✅
```

### Environment Variables
```yaml
env:
  HUGO_VERSION: 0.160.1    # defined once, used in multiple steps
```

Benefits:
```
✅ Single place to update Hugo version
✅ Available to all steps
✅ Easy to maintain
```

### Why `rm -rf public/` Before Build
```yaml
run: |
  rm -rf public/    # always clean build
  hugo --minify
```

Hugo may skip rebuilding unchanged files — stale files cause:
```
❌ Broken links
❌ Wrong asset URLs
❌ Outdated content served
```

Always clean build in CI/CD.

### Why `--minify`
```
hugo --minify:
    → removes whitespace from HTML
    → removes comments
    → compresses CSS/JS
    → smaller file sizes
    → faster page loads
    → only for production ✅
```

### Deploy Flags
```bash
hugo deploy \
  --target=gcs \      # use GCS target from hugo.toml
  --maxDeletes=-1 \   # delete ALL removed files from bucket
  --invalidateCDN     # invalidate Cloud CDN cache after upload
```

`--maxDeletes=-1` means no limit on deletions — ensures removed
pages are cleaned up from the bucket.

`--invalidateCDN` tells Hugo to invalidate Cloud CDN cache after
uploading so visitors immediately see the new version.

---

## Step Separation Best Practice

Each step should do exactly ONE thing:

```
✅ Good — separated steps:
    Step 4: Authenticate to GCP     ← who am I?
    Step 5: Setup gcloud CLI        ← install the tool
    Step 6: Deploy to GCS           ← do the work
    Step 7: Verify                  ← confirm success

❌ Bad — combined steps:
    Step 4: Auth + Setup + Deploy   ← what failed?
```

Benefits of separation:
```
Clearer logs:
    ✅ Authenticate to Google Cloud  (2s)
    ✅ Setup gcloud CLI              (5s)
    ❌ Deploy to GCS                 ← failed here specifically

vs combined:
    ❌ Upload files to GCS           ← where exactly did it fail?
```

---

## Node.js Deprecation Warning

GitHub Actions is deprecating Node.js 20 in favor of Node.js 24.
Add this environment variable to opt into Node.js 24 now:

```yaml
env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
```

This silences the warning and future-proofs the workflow before
the June 2026 deadline.

---

## The Complete Deployment Flow

```
Developer:
    edit content/experience/_index.md
    git add .
    git commit -m "Update work experience"
    git push origin main
            ↓
GitHub:
    detects push to main branch
    triggers deploy.yml workflow
            ↓
GitHub Actions Runner (Ubuntu):
    ✅ Checkout code + PaperMod submodule
    ✅ Install Hugo 0.160.1+extended+withdeploy
    ✅ rm -rf public/ && hugo --minify
    ✅ Authenticate via Workload Identity Federation
    ✅ Setup gcloud CLI
    ✅ hugo deploy --invalidateCDN
    ✅ Deployment complete
            ↓
GCS Bucket (gs://example.com):
    New files uploaded ✅
    Old files removed ✅
    CDN cache invalidated ✅
            ↓
https://example.com:
    Visitors see updated resume immediately ✅
```

---

## Key Concepts Learned

### GitHub Actions
- Runners are fresh Ubuntu VMs per workflow run
- Actions are reusable pre-built steps from marketplace
- Secrets store sensitive values securely
- Version pinning prevents unexpected breakage

### Hugo Installation
- `peaceiris/actions-hugo` doesn't support `+withdeploy`
- Must install Hugo manually from GitHub releases
- Always verify exact filename for your version
- `HUGO_VERSION` env var makes updates easy

### CI/CD Best Practices
- Always clean build (`rm -rf public/`) in pipelines
- Separate steps for clarity and debugging
- Pin all versions (Hugo, Actions) for reproducibility
- Never store credentials in code — use Secrets

---

## Upgrading Hugo in the Future

To upgrade Hugo version in CI/CD:

```yaml
# Change one line in workflow:
env:
  HUGO_VERSION: 0.161.0    # updated version
```

But first verify locally:
```bash
# Test new version on Mac
brew upgrade hugo
hugo server -D    # verify site still works
```

Then update workflow and push.

---

## Current Infrastructure State

```
✅ GitHub Repo:     example/my-resume
✅ Workflow:        .github/workflows/deploy.yml
✅ GitHub Secrets:  GCP_PROJECT_ID, WIF_PROVIDER, SERVICE_ACCOUNT
✅ Pipeline:        triggers on push to main
✅ Hugo Install:    extended+withdeploy from GitHub releases
✅ Authentication:  Workload Identity Federation (passwordless)
✅ Deployment:      hugo deploy → gs://example.com
✅ CDN:             cache invalidated after each deploy
```

---

## Up Next

With CI/CD pipeline working, we do a final end-to-end test
to verify the complete workflow from code change to live site.
That is covered in Part 12.

---

*This post is part of a series on building a Hugo resume site deployed
to Google Cloud Platform with a full CI/CD pipeline via GitHub Actions.*
