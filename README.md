# Mohammed Azam — Resume Site

A personal resume site built with [Hugo](https://gohugo.io) and the
[PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, hosted on
Google Cloud Storage and served via `zamqua.com`.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Static Site Generator | Hugo v0.160.1 (Extended + Deploy) |
| Theme | PaperMod |
| Hosting | Google Cloud Storage |
| CDN / HTTPS | Google Cloud Load Balancer |
| CI/CD | GitHub Actions |
| Domain | zamqua.com |

---

## Project Structure

```
my-resume/
├── content/                  # Resume content in Markdown
│   ├── _index.md             # Homepage — bio and intro
│   ├── experience/_index.md  # Work experience
│   ├── education/_index.md   # Education and certifications
│   ├── skills/_index.md      # Technical skills
│   └── projects/_index.md    # Notable projects
├── layouts/
│   └── _partials/
│       └── home_info.html    # Override — adds social icons to homepage
├── static/                   # Static assets (favicon, images)
├── themes/
│   └── PaperMod/             # Git submodule
└── hugo.toml                 # Site configuration
```

---

## How It Was Built

### 1. Hugo Installation
Hugo Extended edition was installed via Homebrew on macOS (Apple Silicon):

```bash
brew install hugo
hugo version
# hugo v0.160.1+extended+withdeploy darwin/arm64
```

### 2. Project Scaffold
A new Hugo site was created and immediately placed under Git version control:

```bash
hugo new site my-resume
cd my-resume
git init
```

### 3. Theme Installation
PaperMod was added as a Git submodule so it can be updated independently
of the site content:

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive
```

> [!NOTE]
> hugo v0.160.1+extended+withdeploy darwin/arm64 BuildDate=2026-04-08T14:02:42Z
> has a conflict with PaperMod v8.0. We used working master branch.


### 4. Configuration
The site was configured in `hugo.toml` with:
- Site title and base URL
- PaperMod `homeInfoParams` for homepage intro
- Navigation menu (Experience, Skills, Education, Projects)
- Social icons (GitHub, LinkedIn, Email)

### 5. Resume Content
Each resume section was created as a Hugo **section page** using `_index.md`
files. This structure maps directly to clean URLs:

```
content/experience/_index.md  →  zamqua.com/experience/
content/education/_index.md   →  zamqua.com/education/
content/skills/_index.md      →  zamqua.com/skills/
content/projects/_index.md    →  zamqua.com/projects/
```

Content was written in **Markdown** with YAML front matter:

```markdown
---
title: "Work Experience"
layout: "single"
---

## Senior Engineer
**Company** | City | Date – Date
- Achievement one
- Achievement two
```

### 6. Template Override
PaperMod's `home_info.html` partial was overridden to add social icons
to the homepage without duplicating content. The override lives at:

```
layouts/_partials/home_info.html
```

It adds only the social icons footer — PaperMod's `list.html` already
renders the page body from `content/_index.md`:

```html
<footer class="entry-footer">
    {{ partial "social_icons.html" (dict "align" "left") }}
</footer>
```

### 7. Social Icons
GitHub, LinkedIn, and Email icons are configured in `hugo.toml` using
PaperMod's built-in SVG icon system:

```toml
[[params.socialIcons]]
  name = "github"
  url = "https://github.com/zamqua"

[[params.socialIcons]]
  name = "linkedin"
  url = "https://linkedin.com/in/mazam"

[[params.socialIcons]]
  name = "email"
  url = "mailto:mohammed.azam@gmail.com"
```

---

## Local Development

```bash
# Start dev server with live reload
hugo server -D

# Visit in browser
open http://localhost:1313
```

The `-D` flag includes draft pages during development. Hugo rebuilds
the site in milliseconds on every file save.

---

## Building for Production

```bash
hugo
```

This generates the complete static site in the `public/` directory.
The `public/` folder is excluded from Git — it is generated output,
not source code.

---

## Deployment

The site is deployed to Google Cloud Storage with a full CI/CD pipeline.
Every push to the `main` branch automatically:

1. Triggers a GitHub Actions workflow
2. Installs Hugo on the runner
3. Builds the site (`hugo`)
4. Authenticates to GCP via Workload Identity Federation
5. Uploads `public/` to the GCS bucket
6. Invalidates the Cloud CDN cache

See `.github/workflows/deploy.yml` for the full pipeline definition.

---

## Infrastructure

```
zamqua.com
    ↓
Squarespace DNS  →  A record → GCP Load Balancer IP
    ↓
GCP Load Balancer
    • Google-managed SSL certificate (HTTPS)
    • Cloud CDN enabled
    ↓
GCS Bucket (zamqua.com)
    • Static website hosting enabled
    • Public read access
    • Contains built Hugo output
```

---

## Key Hugo Concepts Used

| Concept | Usage |
|---|---|
| `_index.md` | Section landing pages (experience, skills, etc.) |
| Front matter | Page metadata (title, layout, description) |
| Layouts override | Custom `home_info.html` for social icons |
| Git submodule | PaperMod theme management |
| `hugo server -D` | Local development with live reload |
| `hugo` | Production build to `public/` |

---

## Author
**Mohammed Azam**
=======
# my-resume
My resume for "Cloud Resume Challenge"
