---
title: "End to End Testing — Hugo Resume Site on GCP"
author: " "
date: 2026-05-05
draft: false
tags: ["hugo", "gcp", "testing", "cicd", "devops"]
categories: ["Tutorial"]
description: "Final end to end verification of the complete Hugo resume site deployment pipeline — from code change to live site at example.com."
showToc: true
---

This is the final part of a series on building and deploying a personal
resume site using Hugo and Google Cloud Platform. In this post we verify
every component of the system works correctly — from a code change on
the Mac all the way to the live site at `https://example.com`.

---

## What We're Testing

The complete system has many moving parts. End to end testing
verifies they all work together:

```
Code change on Mac
        ↓
Git push to GitHub
        ↓
GitHub Actions pipeline triggers
        ↓
Hugo builds site
        ↓
Workload Identity authenticates to GCP
        ↓
Files uploaded to GCS bucket
        ↓
CDN cache invalidated
        ↓
Visitor sees updated site at example.com ✅
```

---

## Test 1 — Make a Visible Content Change

Make a small change that will be clearly visible on the live site:

```bash
cd ~/Projects/my-resume
nano content/_index.md
```

Add a visible marker:

```markdown
---
title: "Mohammed Azam"
description: "Principal Software Engineer | Architect | Distributed Systems"
---

Principal Engineer/Architect with 15+ years building high-performance,
cloud-native distributed systems...

📍 Coppell, TX

*Last updated: May 2026*    ← visible change to verify deployment
```

---

## Test 2 — Commit and Push

```bash
git add content/_index.md
git commit -m "Test end to end deployment pipeline"
git push origin main
```

---

## Test 3 — Watch Pipeline Execute

Navigate to GitHub Actions:
```
https://github.com/example/my-resume/actions
```

Verify each step completes successfully:

```
Deploy Hugo Site to GCS
    ✅ Checkout repository        (~10s)
    ✅ Setup Hugo                 (~15s)
    ✅ Build site                 (~5s)
    ✅ Authenticate to GCP        (~5s)
    ✅ Setup gcloud CLI           (~5s)
    ✅ Deploy to GCS              (~10s)
    ✅ Verify deployment          (~1s)
─────────────────────────────────────
Total:                            (~51s)
```

---

## Test 4 — Verify Live Site

Once the pipeline completes visit every page:

```
https://example.com              ← homepage updated ✅
https://example.com/experience/  ← links work ✅
https://example.com/education/   ← links work ✅
https://example.com/skills/      ← links work ✅
https://example.com/projects/    ← links work ✅
http://example.com               ← redirects to https ✅
```

---

## Test 5 — Verify HTTPS

```bash
curl -Iv https://example.com 2>&1 | grep -E "SSL|certificate|issuer"
```

Expected output:
```
SSL connection using TLSv1.3
subject: CN=example.com
issuer: CN=GTS CA 1P5, O=Google Trust Services
```

---

## Test 6 — Verify HTTP Redirect

```bash
curl -I http://example.com
```

Expected output:
```
HTTP/1.1 301 Moved Permanently
Location: https://example.com/    ✅
```

---

## Test 7 — Verify CDN Headers

```bash
curl -I https://example.com
```

Look for cache headers:
```
cache-control: max-age=60       ← HTML cache setting ✅
x-goog-stored-content-length:  ← served via GCP ✅
```

---

## Test 8 — Verify GCS Bucket

```bash
# Verify files exist in bucket
gcloud storage ls gs://example.com

# Check index.html was recently updated
gcloud storage ls -l gs://example.com/index.html
```

---

## The Complete System Architecture

```
┌─────────────────────────────────────────────────┐
│                  YOUR MACBOOK                   │
│  edit content → git push → GitHub              │
└────────────────────────┬────────────────────────┘
                         │ git push
                         ▼
┌─────────────────────────────────────────────────┐
│                    GITHUB                       │
│  Repository: example/my-resume                   │
│                                                 │
│  GitHub Actions Pipeline:                       │
│  1. Checkout code + PaperMod submodule          │
│  2. Install Hugo 0.160.1+extended+withdeploy    │
│  3. hugo --minify                               │
│  4. Workload Identity → GCP token               │
│  5. hugo deploy → gs://example.com               │
│  6. Invalidate CDN cache                        │
└────────────────────────┬────────────────────────┘
                         │ deploys
                         ▼
┌─────────────────────────────────────────────────┐
│           GOOGLE CLOUD PLATFORM                 │
│                                                 │
│  Workload Identity Federation                   │
│  github-actions@my-resume-123456.iam...         │
│                    ↓                            │
│  GCS Bucket: gs://example.com                    │
│  ├── index.html                                 │
│  ├── experience/index.html                      │
│  ├── education/index.html                       │
│  ├── skills/index.html                          │
│  └── projects/index.html                        │
│                    ↓                            │
│  Cloud CDN (edge cache globally)                │
│                    ↓                            │
│  Cloud Load Balancer (12.34.567.89)             │
│  ├── HTTPS proxy + SSL certificate              │
│  ├── HTTP → HTTPS redirect (301)                │
│  └── URL map → backend bucket → GCS            │
└────────────────────────┬────────────────────────┘
                         │ DNS
                         ▼
┌─────────────────────────────────────────────────┐
│              SQUARESPACE DNS                    │
│  example.com    A      12.34.567.89              │
│  www           CNAME  example.com.               │
└────────────────────────┬────────────────────────┘
                         │ serves
                         ▼
┌─────────────────────────────────────────────────┐
│                  VISITOR                        │
│         https://example.com 🎉                   │
└─────────────────────────────────────────────────┘
```

---

## Complete Project Summary

### All Steps Completed

```
✅ Step 1:  Hugo installed (v0.160.1+extended+withdeploy)
✅ Step 2:  Project created (my-resume)
✅ Step 3:  PaperMod theme (pinned to specific commit)
✅ Step 4:  Resume content (created from PDF)
✅ Step 5:  Pushed to GitHub (example/my-resume)
✅ Step 6:  GCP project setup (my-resume-123456)
✅ Step 7:  GCS bucket (gs://example.com)
✅ Step 8:  Load Balancer + SSL + CDN
✅ Step 9:  DNS configuration (example.com → LB IP)
✅ Step 10: IAM + Workload Identity Federation
✅ Step 11: GitHub Actions CI/CD pipeline
✅ Step 12: End to end testing ✅
```

### Skills Acquired

**Hugo**
- Static site generation concepts
- Content organization with Markdown and front matter
- Theme installation as Git submodule
- Template override system
- Hugo deploy with GCS backend
- Local development with live reload

**Google Cloud Platform**
- Project and billing configuration
- GCS bucket creation and public access
- Org policies and tag-based security
- Global Load Balancer architecture
- Cloud CDN and cache invalidation
- Google-managed SSL certificates
- IAM service accounts and permissions
- Workload Identity Federation

**DNS and Security**
- DNS record types (A, CNAME, TXT, MX, CAA)
- SSL certificate chain of trust
- HTTP to HTTPS redirect
- Principle of least privilege
- Passwordless CI/CD authentication

**GitHub and CI/CD**
- Git submodules
- GitHub Actions workflow syntax
- GitHub Secrets management
- Personal Access Token scopes
- CI/CD pipeline design and best practices

---

## What to Do Next

### Short Term
```
1. Add more blog posts documenting your learning
2. Add a profile photo to the homepage
3. Customize PaperMod colors and fonts
4. Add Google Analytics
```

### Medium Term (Cost Saving)
```
Replace Load Balancer with Cloudflare:
    Current cost:  ~$18-20/month (Load Balancer)
    After:         ~$0/month (Cloudflare free tier)

Steps:
1. Sign up at cloudflare.com
2. Add example.com to Cloudflare
3. Update nameservers at Squarespace
4. Add CNAME: example.com → c.storage.googleapis.com
5. Enable Cloudflare SSL (one toggle)
6. Delete GCP Load Balancer components
```

### Long Term
```
1. Add contact form (Cloud Functions backend)
2. Add resume PDF download
3. Set up staging environment
4. Add uptime monitoring
5. Implement HSTS for enhanced security
```

---

## Lessons Learned

### Always Clean Build in CI/CD
```bash
rm -rf public/    # before every hugo build
```
Stale files cause broken links and wrong URLs.

### Pin Everything
```
Hugo version:      0.160.1 (in workflow env var)
PaperMod commit:   c4ca7ca (git submodule)
Action versions:   @v4, @v2 (not @main)
```
Pinning ensures reproducible builds forever.

### Separate CI/CD Steps
Each step should do one thing — makes debugging easy
when something fails.

### Use Workload Identity Over Service Account Keys
```
Service Account Key:  permanent, must rotate, can leak ❌
Workload Identity:    temporary, auto-expires, no keys ✅
```

### Org Policies Need Attention
Google Workspace organizations enforce security policies
that can block public GCS access. Plan for this when
working in an org context.

### baseURL Must Match Deployment Target
```
Local dev:    hugo server (baseURL ignored)
GCS direct:   hugo --baseURL="https://storage.googleapis.com/example.com/"
Production:   hugo --baseURL="https://example.com/" (in hugo.toml)
```

---

## Final Cost Summary

```
Current setup (with Load Balancer):
    GCS Storage:     ~$0.01/month
    Load Balancer:   ~$18.00/month
    Cloud CDN:       ~$0.01/month
    GitHub Actions:  $0/month (free tier)
    ──────────────────────────────────
    Total:           ~$18/month

After switching to Cloudflare:
    GCS Storage:     ~$0.01/month
    Cloudflare:      $0/month (free tier)
    GitHub Actions:  $0/month (free tier)
    ──────────────────────────────────
    Total:           ~$0.01/month
```

---

*This post concludes the series on building a Hugo resume site
deployed to Google Cloud Platform with a full CI/CD pipeline
via GitHub Actions. The complete source code is available at
[github.com/example/my-resume](https://github.com/example/my-resume).*
