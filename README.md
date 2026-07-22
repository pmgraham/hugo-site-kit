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
