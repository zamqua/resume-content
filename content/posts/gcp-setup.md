---
title: "Setting Up Google Cloud Platform for Hugo Resume Deployment"
author: " "
date: 2026-05-04
draft: false
tags: ["hugo", "gcp", "devops", "cloud"]
categories: ["Tutorial"]
description: "Step by step guide to setting up Google Cloud Platform for deploying a Hugo static site including project configuration, billing, APIs and permissions."
showToc: true
---

This is Part 6 of a series on building and deploying a personal resume
site using Hugo and Google Cloud Platform. In this post we cover setting
up GCP from scratch — installing the CLI, configuring the project,
linking billing, and enabling required APIs.

---

## Prerequisites

Before starting this step you should have:

- A Google account (Workspace or personal Gmail)
- A GCP project created at `console.cloud.google.com`
- Homebrew installed on macOS

---

## Install the Google Cloud CLI

The `gcloud` CLI is the primary tool for managing GCP resources from
your terminal. Install it via Homebrew on macOS:

```bash
brew install --cask google-cloud-sdk
```

After installation, reload your shell:

```bash
source "$(brew --prefix)/share/google-cloud-sdk/path.bash.inc"
```

Verify the installation:

```bash
gcloud version
```

---

## Authenticate to GCP

```bash
gcloud auth login
```

This opens your browser. Sign in with your Google account. On success
you'll see:

```
You are now logged in as [user@example.com]
```

Also set up Application Default Credentials (ADC) — used by SDKs
and some gcloud commands:

```bash
gcloud auth application-default login
```

---

## Understanding GCP Project Identifiers

Every GCP project has three identifiers. It is important to understand
the difference between them:

| Identifier | Example | Notes |
|---|---|---|
| **Project Name** | `my-resume` | Human readable, can change, not unique |
| **Project ID** | `my-resume-123456` | Globally unique, permanent, use this in CLI |
| **Project Number** | `12345678900` | Numeric, used internally by Google |

Always use the **Project ID** in gcloud commands:

```bash
gcloud config set project my-resume-123456
```

> When you name a project, GCP auto-appends a number if the name is
> already taken globally — hence `my-resume-123456` instead of
> `my-resume`.

---

## Working Under a Google Workspace Organization

If your GCP account is part of a Google Workspace organization, you may
encounter permission issues that personal Gmail accounts don't face.
This is because organizations enforce **org-level policies** that
override project-level permissions.

### Check Your Organization

```bash
gcloud organizations list
```

Output:
```
DISPLAY_NAME   ID            DIRECTORY_CUSTOMER_ID
example.com   123456789000  A01oo23ab
```

### Check Your IAM Role at Org Level

```bash
gcloud organizations get-iam-policy 123456789000 \
    --flatten="bindings[].members" \
    --filter="bindings.members:user@example.com" \
    --format="table(bindings.role)"
```

You need at minimum these roles at the **organization level**:

```
roles/resourcemanager.organizationAdmin
roles/billing.admin
roles/serviceusage.serviceUsageAdmin
```

Grant missing roles:

```bash
gcloud organizations add-iam-policy-binding 123456789000 \
    --member="user:user@example.com" \
    --role="roles/billing.admin"

gcloud organizations add-iam-policy-binding 123456789000 \
    --member="user:user@example.com" \
    --role="roles/serviceusage.serviceUsageAdmin"
```

---

## Verify Billing Is Linked

Without a billing account linked to your project, GCP blocks API
enablement even for free services:

```bash
gcloud billing projects describe my-resume-123456
```

You should see `billingEnabled: true`. If not, link your billing account:

```bash
# List billing accounts
gcloud billing accounts list

# Link billing account to project
gcloud billing projects link my-resume-123456 \
    --billing-account=YOUR_BILLING_ACCOUNT_ID
```

Verify all projects under your billing account:

```bash
gcloud billing projects list --billing-account=YOUR_BILLING_ACCOUNT_ID
```

Output:
```
PROJECT_ID          BILLING_ACCOUNT_ID    BILLING_ENABLED
my-resume-123456    0123AB-C4D56E-7890  True
```

---

## Enable Required APIs

GCP services are disabled by default and must be explicitly enabled.
For a Hugo static site deployment we need:

```bash
gcloud services enable \
    storage.googleapis.com \
    compute.googleapis.com \
    iam.googleapis.com \
    iamcredentials.googleapis.com \
    cloudresourcemanager.googleapis.com \
    --project=my-resume-123456
```

Verify APIs are enabled:

```bash
gcloud services list --enabled --project=my-resume-123456 \
    | grep -E "storage|compute|iam"
```

Expected output:
```
compute.googleapis.com               Compute Engine API
iam.googleapis.com                   Identity and Access Management API
iamcredentials.googleapis.com        IAM Service Account Credentials API
storage.googleapis.com               Cloud Storage API
storage-api.googleapis.com           Google Cloud Storage JSON API
storage-component.googleapis.com     Cloud Storage
```

---

## Fix Application Default Credentials Warning

If you see this warning:

```
WARNING: Your active project does not match the quota project
in your local Application Default Credentials file.
```

Fix it by syncing ADC to your project:

```bash
gcloud auth application-default set-quota-project my-resume-123456
```

---

## Set Default Region and Zone

```bash
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
```

`us-central1` is recommended — it is Google's largest region with
the most services available and lowest latency for US visitors.

---

## Final Verification

Run these commands to confirm everything is configured correctly:

```bash
# Active account
gcloud auth list

# Active project
gcloud config get project

# Billing status
gcloud billing projects describe my-resume-123456

# Enabled APIs
gcloud services list --enabled --project=my-resume-123456 \
    | grep -E "storage|compute|iam"
```

---

## What We Learned

### GCP Concepts
- **Project ID** vs **Project Name** vs **Project Number** — always use Project ID in CLI commands
- **Organization policies** — Workspace orgs enforce restrictions that override project permissions
- **Billing account** — must be linked before any APIs can be enabled
- **Application Default Credentials** — separate from user credentials, used by SDKs

### Key Lessons
- Being **Owner** of a project is not enough inside a Workspace org — you also need org-level roles
- Always check `gcloud billing projects describe` first when APIs fail to enable
- The `--project` flag explicitly targets a project regardless of your default config

---

## Up Next

With GCP configured we are ready to create our **Google Cloud Storage
bucket** and configure it for static website hosting. That is covered
in Part 7 of this series.

---

*This post is part of a series on building a Hugo resume site deployed
to Google Cloud Platform with a full CI/CD pipeline via GitHub Actions.*
