# Hugo Site Kit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `hugo-site-kit` GitHub template repo: a Hugo scaffold pre-wired with Sveltia CMS at `/admin`, CI build verification, and setup docs for Cloudflare Pages + the shared OAuth worker.

**Architecture:** The repo is configuration and docs only — no application code. A minimal-but-building Hugo site, a two-file Sveltia admin (`index.html` + `config.yml`), a GitHub Actions build check, and a `SETUP.md` covering the one-time OAuth worker deploy and the per-site setup flow.

**Tech Stack:** Hugo 0.162.1 extended, Sveltia CMS 0.172.3 (pinned, loaded from unpkg — no npm/build step), GitHub Actions, Cloudflare Pages, sveltia-cms-auth Cloudflare Worker (stock, deployed once, not part of this repo).

## Global Constraints

- Repo root: `/Users/pmgraham/projects/hugo-site-kit` (git repo already initialized, branch `main`; spec committed).
- Hugo version pinned everywhere it appears: `0.162.1` (extended).
- Sveltia CMS version pinned in `static/admin/index.html`: `0.172.3`.
- No Node, no npm, no build tooling in the template — Sveltia loads as a single pinned script.
- The template must build cleanly with `hugo --minify` at every commit.
- Themes/styling are out of scope: layouts are deliberately minimal semantic HTML placeholders, intended to be replaced per site (by Claude/Gemini).
- No site-type content presets: `config.yml` ships one example "pages" collection only.
- All work happens on `main` with a commit per task (this repo is a template, not a production service).

---

### Task 1: Hugo scaffold that builds

**Files:**
- Create: `.gitignore`
- Create: `hugo.toml`
- Create: `layouts/baseof.html`
- Create: `layouts/home.html`
- Create: `layouts/page.html`
- Create: `layouts/section.html`
- Create: `archetypes/default.md`
- Create: `content/_index.md`
- Create: `content/about.md`
- Create: `assets/.gitkeep`, `static/uploads/.gitkeep`

**Interfaces:**
- Consumes: nothing (first task).
- Produces: a Hugo site where `hugo --minify` exits 0 and writes `public/index.html` and `public/about/index.html`. Task 2 relies on the `static/` directory being copied verbatim into `public/`, and on `static/uploads/` existing as the media folder. Tasks 3–5 rely on `hugo --minify` being the canonical build command.

- [ ] **Step 1: Write `.gitignore`**

```gitignore
public/
resources/_gen/
.hugo_build.lock
```

- [ ] **Step 2: Write `hugo.toml`**

```toml
# Replace baseURL and title when creating a new site from this template.
baseURL = 'https://example.com/'
languageCode = 'en-us'
title = 'Hugo Site Kit'

disableKinds = ['taxonomy', 'term']

[markup]
  [markup.goldmark]
    [markup.goldmark.renderer]
      unsafe = true
```

- [ ] **Step 3: Write the placeholder layouts**

These are intentionally bare semantic HTML — each new site replaces them during theme work. Hugo ≥0.146 resolves templates from the `layouts/` root (no `_default/` directory).

`layouts/baseof.html`:

```html
<!DOCTYPE html>
<html lang="{{ site.Language.LanguageCode }}">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{{ if not .IsHome }}{{ .Title }} | {{ end }}{{ site.Title }}</title>
</head>
<body>
  <header>
    <nav><a href="{{ site.Home.RelPermalink }}">{{ site.Title }}</a></nav>
  </header>
  <main>
    {{ block "main" . }}{{ end }}
  </main>
</body>
</html>
```

`layouts/home.html`:

```html
{{ define "main" }}
<h1>{{ .Title | default site.Title }}</h1>
{{ .Content }}
<ul>
  {{ range site.RegularPages }}
  <li><a href="{{ .RelPermalink }}">{{ .Title }}</a></li>
  {{ end }}
</ul>
{{ end }}
```

`layouts/page.html`:

```html
{{ define "main" }}
<h1>{{ .Title }}</h1>
{{ .Content }}
{{ end }}
```

`layouts/section.html`:

```html
{{ define "main" }}
<h1>{{ .Title }}</h1>
{{ .Content }}
<ul>
  {{ range .Pages }}
  <li><a href="{{ .RelPermalink }}">{{ .Title }}</a></li>
  {{ end }}
</ul>
{{ end }}
```

- [ ] **Step 4: Write archetype and example content**

`archetypes/default.md`:

```markdown
+++
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
date = '{{ .Date }}'
draft = true
+++
```

`content/_index.md`:

```markdown
+++
title = 'Home'
+++

Welcome. This site was created from **hugo-site-kit**. Replace this text
via the content manager at `/admin`.
```

`content/about.md`:

```markdown
+++
title = 'About'
date = 2026-07-22T00:00:00-00:00
draft = false
+++

An example page managed through the CMS. Edit or delete it at `/admin`.
```

- [ ] **Step 5: Create the empty tracked directories**

Run:

```bash
mkdir -p assets static/uploads && touch assets/.gitkeep static/uploads/.gitkeep
```

- [ ] **Step 6: Verify the build fails-then-passes check**

Run: `cd /Users/pmgraham/projects/hugo-site-kit && hugo --minify`
Expected: exit 0, output table showing Pages ≥ 3, no `WARN found no layout file` lines.

Run: `test -f public/index.html && test -f public/about/index.html && echo BUILD_OK`
Expected: `BUILD_OK`

- [ ] **Step 7: Commit**

```bash
git add .gitignore hugo.toml layouts archetypes content assets static
git commit -m "feat: Hugo scaffold with placeholder layouts and example content"
```

---

### Task 2: Sveltia CMS admin

**Files:**
- Create: `static/admin/index.html`
- Create: `static/admin/config.yml`

**Interfaces:**
- Consumes: Task 1's `static/` passthrough and `static/uploads/` media folder; `content/` as the pages collection folder.
- Produces: `public/admin/index.html` and `public/admin/config.yml` in build output. `SETUP.md` (Task 4) tells the site creator to edit exactly two values in `config.yml`: `backend.repo` and `backend.base_url`.

- [ ] **Step 1: Compute the SRI hash for the pinned Sveltia bundle**

The admin page handles GitHub auth tokens, so the script tag must carry a
Subresource Integrity hash — a compromised CDN must not be able to swap
the bundle.

Run:

```bash
curl -sSL https://unpkg.com/@sveltia/cms@0.172.3/dist/sveltia-cms.js | openssl dgst -sha384 -binary | openssl base64 -A
```

Expected: a base64 string (~64 chars). Use it as `<HASH>` in the next step.

- [ ] **Step 2: Write `static/admin/index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Content Manager</title>
  <link rel="icon" href="data:," />
</head>
<body>
  <!-- Sveltia CMS, pinned + SRI. To upgrade: bump the version AND recompute
       the integrity hash:
       curl -sSL https://unpkg.com/@sveltia/cms@<ver>/dist/sveltia-cms.js | openssl dgst -sha384 -binary | openssl base64 -A -->
  <script src="https://unpkg.com/@sveltia/cms@0.172.3/dist/sveltia-cms.js"
          integrity="sha384-<HASH>"
          crossorigin="anonymous"
          type="module"></script>
</body>
</html>
```

Replace `sha384-<HASH>` with the value from Step 1.

- [ ] **Step 3: Write `static/admin/config.yml`**

```yaml
# Sveltia CMS configuration.
# Per-site setup (see SETUP.md): change `repo` and `base_url` below,
# then replace the example collection with this site's content model.

backend:
  name: github
  repo: OWNER/REPO # ← your GitHub org/repo, e.g. pmgraham/family-business-site
  branch: main
  base_url: https://sveltia-cms-auth.YOUR-SUBDOMAIN.workers.dev # ← your shared OAuth worker URL

media_folder: static/uploads
public_folder: /uploads

collections:
  - name: pages
    label: Pages
    folder: content
    create: true
    extension: md
    format: toml-frontmatter
    fields:
      - { label: Title, name: title, widget: string }
      - { label: Date, name: date, widget: datetime, required: false }
      - { label: Draft, name: draft, widget: boolean, default: false }
      - { label: Body, name: body, widget: markdown }
```

- [ ] **Step 4: Verify the admin ships in the build**

Run: `cd /Users/pmgraham/projects/hugo-site-kit && hugo --minify && test -f public/admin/index.html && test -f public/admin/config.yml && echo ADMIN_OK`
Expected: `ADMIN_OK`

Run: `ruby -ryaml -e 'YAML.load_file("static/admin/config.yml"); puts "YAML_OK"'`
Expected: `YAML_OK`

- [ ] **Step 5: Commit**

```bash
git add static/admin
git commit -m "feat: Sveltia CMS admin with pinned version, SRI, and example collection"
```

---

### Task 3: CI build check

**Files:**
- Create: `.github/workflows/build.yml`

**Interfaces:**
- Consumes: Task 1's `hugo --minify` build contract.
- Produces: a `build` workflow that fails any push/PR where the template (or a site made from it) stops building. `README.md` (Task 5) references this workflow by the name `build`.

- [ ] **Step 1: Write `.github/workflows/build.yml`**

```yaml
name: build

on:
  push:
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.162.1'
          extended: true

      - name: Build site
        run: hugo --minify
```

- [ ] **Step 2: Verify the workflow YAML parses**

Run: `ruby -ryaml -e 'YAML.load_file(".github/workflows/build.yml"); puts "YAML_OK"'`
Expected: `YAML_OK`

(The workflow itself runs on GitHub after the repo is pushed — verified in Task 5's final check.)

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/build.yml
git commit -m "ci: verify Hugo build on every push"
```

---

### Task 4: SETUP.md — one-time worker setup and per-site walkthrough

**Files:**
- Create: `SETUP.md`

**Interfaces:**
- Consumes: file paths and the two `config.yml` placeholders (`backend.repo`, `backend.base_url`) exactly as defined in Task 2.
- Produces: the complete operator manual. `README.md` (Task 5) links to it.

- [ ] **Step 1: Write `SETUP.md` with the following content**

````markdown
# Setup Guide

Two parts: a **one-time** OAuth worker deploy (first site only, ~10
minutes), then the **per-site** steps you repeat for every new site
(~1 hour, most of it content modeling).

## One-time: deploy the shared OAuth worker

Sveltia CMS needs a tiny server-side helper to complete GitHub sign-in.
You deploy it **once** on your Cloudflare account; every site you ever
create from this template shares it.

1. **Deploy the worker.** Go to
   <https://github.com/sveltia/sveltia-cms-auth> and click the
   **Deploy to Cloudflare Workers** button (or clone and `wrangler deploy`).
   Note the worker URL, e.g. `https://sveltia-cms-auth.<your-subdomain>.workers.dev`.
2. **Register a GitHub OAuth app.** GitHub → Settings → Developer
   settings → OAuth Apps → New OAuth App:
   - Application name: anything (e.g. `My Sites CMS`)
   - Homepage URL: the worker URL
   - Authorization callback URL: `<worker URL>/callback`
   Then generate a client secret.
3. **Configure the worker.** In the Cloudflare dashboard → your worker →
   Settings → Variables, add:
   - `GITHUB_CLIENT_ID` — from the OAuth app
   - `GITHUB_CLIENT_SECRET` — from the OAuth app (encrypt it)
   - `ALLOWED_DOMAINS` — comma-separated list of site domains allowed to
     use this worker, e.g. `mybusiness.com,artportfolio.com`. You will
     append to this list for every new site.

## Per-site: create a new site

### 1. Create the repo

On this template repo's GitHub page, click **Use this template** →
**Create a new repository**. Name it after the site.

### 2. Point Cloudflare Pages at it

Cloudflare dashboard → Workers & Pages → Create → Pages → Connect to Git
→ pick the new repo:

- Build command: `hugo --minify`
- Build output directory: `public`
- Environment variable: `HUGO_VERSION` = `0.162.1`

First deploy gives you `<project>.pages.dev`; add the real custom domain
under the project's **Custom domains** tab.

### 3. Wire up the CMS

In the new repo, edit `static/admin/config.yml`:

- `backend.repo`: the new repo, e.g. `pmgraham/family-business-site`
- `backend.base_url`: your shared worker URL from the one-time setup

Then append the site's domain (and its `<project>.pages.dev` domain if
you want admin access there too) to the worker's `ALLOWED_DOMAINS`
variable in the Cloudflare dashboard.

Also update `hugo.toml`: set `baseURL` and `title`.

### 4. Invite the editor

The editor needs a free GitHub account. Repo → Settings → Collaborators
→ invite their username (Write access). They accept the email invite
once; after that they never see GitHub again — they use
`https://<site domain>/admin`.

### 5. Define the content model

Replace the example `pages` collection in `static/admin/config.yml`
with this site's real content structure. Describe the site to an AI
assistant using this prompt:

> Here is my current `static/admin/config.yml` for a Hugo site using
> Sveltia CMS (GitHub backend, media in `static/uploads`). Rewrite the
> `collections` section for this site: [describe it — e.g. "an art
> portfolio: each Work has a title, one image, medium, dimensions, year,
> and an optional description; plus an About page and a Contact page"].
> Keep the backend and media settings unchanged. Use folder collections
> for repeating content and file collections for one-off pages. Match
> the front matter to Hugo conventions (`toml-frontmatter`, `title`,
> `date`, `draft`).

Create matching folders under `content/` and, if needed, archetypes.
Commit, push, and verify the collections render at `/admin`.

### 6. Smoke-test the loop

1. Open `https://<site domain>/admin` in a private window.
2. Sign in with the **editor's** GitHub account.
3. Edit a page, save.
4. Confirm the commit appears in the repo and Cloudflare redeploys
   (~1 minute), and the change is live.

## Troubleshooting

- **Sign-in popup errors or closes immediately** — the site's domain is
  missing from the worker's `ALLOWED_DOMAINS`, or the OAuth callback URL
  doesn't match `<worker URL>/callback`.
- **Editor gets "repository not found"** — the collaborator invite
  wasn't accepted, or `backend.repo` points at the wrong repo.
- **Save works but the site doesn't update** — check the Cloudflare
  Pages build log; a bad field value can break the build. The public
  site keeps serving the last good deploy while you fix it.
- **`/admin` is blank** — check the browser console; usually a YAML
  syntax error in `config.yml`.
````

- [ ] **Step 2: Verify the doc's fenced blocks and links are intact**

Run: `grep -c 'ALLOWED_DOMAINS' SETUP.md`
Expected: `3` (defined once, referenced in per-site step and troubleshooting).

- [ ] **Step 3: Commit**

```bash
git add SETUP.md
git commit -m "docs: one-time worker setup and per-site walkthrough"
```

---

### Task 5: README, template polish, and publish

**Files:**
- Create: `README.md`
- Modify: none

**Interfaces:**
- Consumes: `SETUP.md` (Task 4), the `build` workflow name (Task 3).
- Produces: the published GitHub template repo, ready to stamp out the first real site.

- [ ] **Step 1: Write `README.md`**

```markdown
# hugo-site-kit

A GitHub template for Hugo sites that non-technical editors can manage.

- **Hugo** generates the site — fully static, no server.
- **Sveltia CMS** provides friendly editing forms at `yoursite.com/admin`;
  saves become git commits. No install, no build step.
- **Cloudflare Pages** rebuilds and deploys on every commit (~1 minute).
- A tiny shared **OAuth worker** (deployed once, ever) handles GitHub
  sign-in for all your sites.

Editors never see git, front matter, or a terminal.

## Creating a new site

Click **Use this template**, then follow [SETUP.md](SETUP.md).
About an hour per site; the one-time OAuth worker setup adds ~10 minutes
to your first site.

## What's in the box

| Path | Purpose |
| --- | --- |
| `hugo.toml` | Minimal Hugo config — set `baseURL` and `title` per site |
| `layouts/` | Bare placeholder templates, meant to be replaced per site |
| `content/` | Example home + about pages |
| `static/admin/` | Sveltia CMS (pinned version) + `config.yml` content model |
| `static/uploads/` | Media folder (CMS image uploads land here) |
| `.github/workflows/build.yml` | `build` check: `hugo --minify` on every push |
| `SETUP.md` | One-time and per-site setup walkthrough |

## Local preview

```bash
hugo server
```

Requires Hugo extended ≥ 0.162.1.

## Design docs

- [Design spec](docs/superpowers/specs/2026-07-22-hugo-site-kit-design.md)
```

- [ ] **Step 2: Verify final build from a clean slate**

Run: `cd /Users/pmgraham/projects/hugo-site-kit && rm -rf public && hugo --minify && test -f public/admin/config.yml && echo FINAL_OK`
Expected: `FINAL_OK`

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: README for the template repo"
```

- [ ] **Step 4: Publish to GitHub and mark as template** *(needs user's gh auth)*

```bash
gh repo create hugo-site-kit --public --source . --push
gh repo edit --template
```

Expected: repo visible on GitHub with the **Use this template** button; the `build` action runs green on the pushed commits (check with `gh run watch` or `gh run list --limit 1`).

If `gh` is not authenticated, stop and ask the user to run `gh auth login` (or create the repo manually) — do not skip the template flag.
