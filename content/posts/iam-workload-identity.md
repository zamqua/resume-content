---
title: "IAM and Workload Identity Federation for Secure CI/CD"
author: " "
date: 2026-05-05
draft: false
tags: ["hugo", "gcp", "iam", "security", "cicd", "workload-identity"]
categories: ["Tutorial"]
description: "Step by step guide to setting up GCP IAM Service Accounts and Workload Identity Federation for secure passwordless authentication between GitHub Actions and Google Cloud Platform."
showToc: true
---

This is Part 10 of a series on building and deploying a personal resume
site using Hugo and Google Cloud Platform. In this post we cover setting
up secure authentication between GitHub Actions and GCP using Workload
Identity Federation — the modern passwordless approach recommended by
Google.

---

## The Problem We're Solving

GitHub Actions runs OUTSIDE of GCP. To deploy files to GCS it needs
GCP credentials. The question is — how do you give GitHub Actions
access to GCP securely?

---

## Two Approaches — Old vs New

### Old Approach — Service Account Key ❌

```
Create service account
        ↓
Download JSON key file
        ↓
Store in GitHub Secrets
        ↓
GitHub Actions uses key to authenticate
```

Problems:
```
❌ Key never expires — permanent security risk
❌ Must be manually rotated periodically
❌ Can be leaked if GitHub is compromised
❌ Discouraged by Google
```

### New Approach — Workload Identity Federation ✅

```
GitHub Actions presents JWT token
        ↓
GCP verifies token came from YOUR repo
        ↓
GCP issues temporary credentials (1 hour)
        ↓
GitHub Actions deploys using temp credentials
        ↓
Credentials automatically expire
```

Benefits:
```
✅ No keys to store or rotate
✅ Credentials expire automatically
✅ Tied to specific GitHub repository
✅ Google's recommended approach
✅ Most secure option available
```

---

## Understanding Workload Identity Federation

### How It Works

GitHub Actions issues a JWT (JSON Web Token) when a workflow runs:

```json
{
  "iss": "token.actions.githubusercontent.com",
  "sub": "repo:example/my-resume:ref:refs/heads/main",
  "actor": "example",
  "repository": "example/my-resume",
  "ref": "refs/heads/main",
  "workflow": "deploy"
}
```

GCP validates this token and exchanges it for temporary credentials.

### The Three Components

```
1. Workload Identity Pool
   Container for trusted external identity providers
   "A trusted group of external systems GCP accepts tokens from"

2. Workload Identity Provider
   Represents GitHub specifically with validation rules
   "GitHub's OIDC endpoint, only for example/my-resume"

3. Service Account
   The GCP identity GitHub Actions impersonates
   "The GCP user GitHub becomes after proving identity"
```

### Real World Analogy

```
Old way (Service Account Key):
    GCP gives GitHub a master key
    Key works forever
    If stolen → permanent access ❌

New way (Workload Identity):
    GitHub shows passport (JWT token)
    GCP verifies passport is genuine
    GCP issues temporary visitor pass (1 hour)
    If stolen → useless after 1 hour ✅
```

---

## What We Created

### 1. Service Account

```bash
gcloud iam service-accounts create github-actions \
    --display-name="GitHub Actions Deploy" \
    --description="Used by GitHub Actions to deploy Hugo site to GCS" \
    --project=$PJ
```

This creates the GCP identity that GitHub Actions will act as:
```
github-actions@my-resume-123456.iam.gserviceaccount.com
```

### 2. Bucket Level IAM Permission

```bash
gcloud storage buckets add-iam-policy-binding gs://example.com \
    --member="serviceAccount:github-actions@my-resume-123456.iam.gserviceaccount.com" \
    --role="roles/storage.objectAdmin"
```

**Why bucket level not project level?**

```
Project level IAM:
    → applies to ALL resources in project
    → too broad ❌

Bucket level IAM:
    → applies ONLY to gs://example.com
    → principle of least privilege ✅
```

`roles/storage.objectAdmin` grants:
```
✅ Upload files (hugo deploy needs this)
✅ Delete files (hugo deploy cleanup)
✅ List files (hugo deploy diff check)
❌ Delete bucket (not granted — too dangerous)
❌ Change bucket settings (not granted)
```

> 💡 To verify bucket level permissions go to:
> GCS Console → example.com bucket → Permissions tab
> NOT the main IAM console (which only shows project level)

### 3. CDN Cache Invalidation Permission

```bash
gcloud projects add-iam-policy-binding $PJ \
    --member="serviceAccount:github-actions@my-resume-123456.iam.gserviceaccount.com" \
    --role="roles/compute.networkAdmin"
```

Needed to invalidate Cloud CDN cache after deployment so
visitors see the new version immediately.

### 4. Workload Identity Pool

```bash
gcloud iam workload-identity-pools create github-pool \
    --location="global" \
    --display-name="GitHub Actions Pool" \
    --description="Identity pool for GitHub Actions" \
    --project=$PJ
```

A container that groups external identity providers. Think of it
as a trusted group of external systems GCP will accept tokens from.

> ⚠️ "Pool" here is a security concept — NOT a thread pool.
> It has nothing to do with processing requests concurrently.

### 5. Workload Identity Provider

```bash
gcloud iam workload-identity-pools providers create-oidc github-provider \
    --location="global" \
    --workload-identity-pool=github-pool \
    --display-name="GitHub Provider" \
    --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
    --attribute-condition="assertion.repository=='example/my-resume'" \
    --issuer-uri="https://token.actions.githubusercontent.com" \
    --project=$PJ
```

Key flags:

| Flag | Purpose |
|---|---|
| `--issuer-uri` | GitHub's OIDC token endpoint |
| `--attribute-mapping` | Maps GitHub JWT claims to GCP attributes |
| `--attribute-condition` | Only allows YOUR specific repo |

**Why the condition matters:**
```
Without condition:
    ANY GitHub repo could impersonate service account ❌

With condition (assertion.repository=='example/my-resume'):
    ONLY example/my-resume repo can impersonate ✅
```

### 6. Service Account Impersonation Binding

```bash
gcloud iam service-accounts add-iam-policy-binding \
    github-actions@my-resume-123456.iam.gserviceaccount.com \
    --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/example/my-resume" \
    --role="roles/iam.workloadIdentityUser" \
    --project=$PJ
```

This creates the link between Workload Identity and Service Account.

>[!NOTE] The principalSet can be created manaully or retrieve it from the console (IAM/WorkloadIdentityPool/github-pool-name/IAM principal)


**Understanding the `--member` value:**

```
principalSet://
    "A SET of principals matching a condition"

iam.googleapis.com
    "Managed by GCP IAM"

/projects/515770216189
    "In your GCP project (using project NUMBER)"

/locations/global
    "Global location (WIF is always global)"

/workloadIdentityPools/github-pool
    "Inside github-pool"

/attribute.repository/example/my-resume
    "Where attribute repository = example/my-resume"

Plain English:
"Any GitHub Actions workflow running in
 the example/my-resume repository"
```

---

## Understanding JWT Token Attributes

When choosing how to restrict access, GitHub's JWT token
provides several attributes:

| Attribute | Filters by | Best for |
|---|---|---|
| `actor` | GitHub username | User specific access |
| `repository` | Repository name | Repo level access ✅ |
| `subject` | Full workflow path + branch | Most restrictive |

We used `repository` — any workflow in `example/my-resume`
can deploy. For extra security use `subject` to restrict
to main branch only:

```
sub = "repo:example/my-resume:ref:refs/heads/main"
→ only main branch can deploy
→ feature branches cannot deploy
```

---

## The Complete Authentication Flow

```
1. git push to example/my-resume main branch
        ↓
2. GitHub Actions workflow triggers
        ↓
3. GitHub issues JWT token:
   {
     "repository": "example/my-resume",
     "ref": "refs/heads/main"
   }
        ↓
4. Workflow requests GCP credentials using JWT
        ↓
5. GCP Workload Identity Provider validates:
   "Token from token.actions.githubusercontent.com?" ✅
   "Repository is example/my-resume?" ✅
        ↓
6. GCP issues temporary token for:
   github-actions@my-resume-123456.iam.gserviceaccount.com
        ↓
7. Workflow uses token to:
   → hugo deploy → uploads to gs://example.com ✅
   → invalidate CDN cache ✅
        ↓
8. Token expires after 1 hour automatically ✅
```

---

## IAM Concepts Learned

### Principle of Least Privilege
Grant only the minimum permissions needed:

```
github-actions needs:
    Write to gs://example.com ✅ granted at bucket level

github-actions does NOT need:
    Write to other buckets ❌ not granted
    Delete buckets ❌ not granted
    Modify project settings ❌ not granted
```

### Two Levels of IAM in GCP

```
Project Level IAM:
    console → IAM & Admin → IAM
    applies to ALL resources in project

Resource Level IAM:
    set directly on resource (bucket, topic, etc.)
    applies ONLY to that specific resource
    NOT shown in main IAM console
```

### Application Default Credentials (ADC)

For local development `hugo deploy` uses ADC:

```
gcloud auth login
    → credentials for gcloud CLI commands

gcloud auth application-default login
    → credentials for applications (hugo deploy)
```

Both expire — CI/CD uses Service Accounts instead which
never require human re-authentication.

---

## Values Needed for Step 11

```bash
# Service Account Email
github-actions@my-resume-123456.iam.gserviceaccount.com

# Workload Identity Provider Resource Name
gcloud iam workload-identity-pools providers describe github-provider \
    --location="global" \
    --workload-identity-pool=github-pool \
    --project=$PJ \
    --format="get(name)"

# Output:
# projects/515770216189/locations/global/workloadIdentityPools/github-pool/providers/github-provider

# Project ID
my-resume-123456
```

---

## Current Infrastructure State

```
✅ Service Account:     github-actions@my-resume-123456.iam.gserviceaccount.com
✅ Bucket Permission:   objectAdmin on gs://example.com
✅ CDN Permission:      compute.networkAdmin on project
✅ Identity Pool:       github-pool (global)
✅ Identity Provider:   github-provider (OIDC, GitHub)
✅ Impersonation:       example/my-resume → github-actions@...
```

---

## Up Next
1G
With IAM and Workload Identity configured, we are ready to
create the **GitHub Actions CI/CD pipeline** that automatically
builds and deploys the Hugo site on every push to main.
That is covered in Part 11.

---

*This post is part of a series on building a Hugo resume site deployed
to Google Cloud Platform with a full CI/CD pipeline via GitHub Actions.*
