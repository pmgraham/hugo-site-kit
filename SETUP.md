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
