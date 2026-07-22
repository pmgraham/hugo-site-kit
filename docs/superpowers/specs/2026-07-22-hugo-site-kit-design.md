# Hugo Site Kit — Design

**Date:** 2026-07-22
**Status:** Approved

## Purpose

A GitHub template repository that stamps out new Hugo sites pre-wired for
non-technical content editing via Sveltia CMS, deployed on Cloudflare Pages.
The goal: a non-technical editor (family member running a business site, a
friend managing an art portfolio) can add and manage content through friendly
web forms, while Hugo remains the static site generator and the site itself
stays fully static.

The kit is configuration, conventions, and documentation — not software.
There is no custom application to maintain.

## Hard requirements

- Hugo generates the site; the published site is fully static.
- Hosted on Cloudflare Pages with minimal setup.
- Non-technical editors manage content without touching git, front matter,
  or a terminal.
- The kit is a plain scaffold: each project defines its own content
  structure at setup time (no baked-in site-type presets).

## Architecture

Four pieces — three per site, one shared across all sites:

### 1. Site repo (per site, created from the template)

A standard Hugo skeleton:

- `hugo.toml` — minimal config with placeholders for site name/base URL.
- `content/`, `archetypes/`, `assets/`, `static/` — standard Hugo layout.
- `static/admin/index.html` — loads Sveltia CMS from a pinned version of
  its single JS bundle. No build step, no npm.
- `static/admin/config.yml` — the site's content model (collections,
  fields, media settings). Ships as a minimal working example (a "pages"
  collection) intended to be replaced per project.
- `SETUP.md` — the per-site setup walkthrough.
- CI workflow — runs `hugo --minify` on every push so breakage is caught
  even outside Cloudflare.

### 2. Sveltia CMS (no install)

Editors browse to `theirsite.com/admin`, sign in with GitHub, and edit
through forms generated from `config.yml`. Saves become commits to the
repo. Images upload through the same UI into the repo's media folder.
The Sveltia version is pinned in `index.html` and upgraded deliberately.

### 3. OAuth worker (shared — deployed once, ever)

Sveltia's stock [sveltia-cms-auth](https://github.com/sveltia/sveltia-cms-auth)
Cloudflare Worker handles the GitHub OAuth handshake. One deployment on the
kit owner's Cloudflare account serves every site; each new site is added to
the worker's allowed-origins list and points its `config.yml` at the worker
URL. This is the only infrastructure in the system.

### 4. Cloudflare Pages (per site)

Watches the site repo, runs `hugo` on every push, serves the static output.
Editor save → commit → auto rebuild → live in about a minute. Deploy
configuration is kept to a few obvious lines so a future migration to
Cloudflare Workers static assets (Cloudflare's stated direction) is trivial.

## Per-site setup flow (documented in SETUP.md)

1. "Use this template" → new site repo.
2. Create a Cloudflare Pages project pointed at the repo (Hugo preset).
3. Add the new site's origin to the shared OAuth worker's allowed list.
4. Invite the editor's GitHub account as a repo collaborator.
5. Define the content model in `static/admin/config.yml` — the per-project
   "add structure" step. An AI assistant (Claude/Gemini) can translate a
   plain-language description ("each work has a title, image, medium,
   year") into collection config; SETUP.md includes a prompt template
   for this.

Target: under an hour per new site, most of it content modeling.

One-time prerequisite (first site only): register a GitHub OAuth app and
deploy the sveltia-cms-auth worker (~10 minutes).

## Editor experience

- Go to `theirsite.com/admin`, sign in with GitHub (one-time collaborator
  invite accepted beforehand).
- See only the collections defined for their site, as plain forms with
  labeled fields and image upload.
- Save publishes: commit → Cloudflare rebuild → live in ~1 minute.
- Editors never see git, markdown front matter, or a terminal.

## Failure modes

- **Editor save fails** (token expiry, conflict — rare): Sveltia surfaces
  the error; the draft stays in the browser.
- **Build fails** (bad front matter/content): Cloudflare keeps serving the
  last good deploy — the public site never breaks; the build log identifies
  the problem.
- **OAuth worker down**: editing is blocked; published sites are unaffected
  (serving never depends on the worker).

## Testing

- The template repo is itself a working demo site.
- CI runs `hugo --minify` on every push to the template, so template
  changes never ship broken.
- The editing loop (sign in at `/admin`, edit, save, rebuild) is verified
  end-to-end on the template's own deployment after any kit change.

## Out of scope

- Themes and styling (handled per site with Claude/Gemini, separately).
- Custom editor UI or any server-side code beyond the stock OAuth worker.
- Multi-language sites.
- Site-type content presets (each project defines its own model).
