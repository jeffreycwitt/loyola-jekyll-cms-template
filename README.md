# loyola-jekyll-cms-template

A reusable template system for publishing Jekyll static sites from a [Payload CMS](https://payloadcms.com) instance. A pre-processing script fetches posts from the Payload REST API, converts the Lexical rich-text content to HTML, and writes standard Jekyll post files. GitHub Actions builds and deploys the site to GitHub Pages on a schedule and on every push.

---

## Architecture

```
loyola-jekyll-cms-template/   ← this repo (template + themes + build logic)
loyola-jekyll-site-2/         ← example site repo (secrets + theme selection only)
loyola-jekyll-site-N/         ← any number of additional site repos
```

**All development happens in this template repo.** Site repos are fully automated and never edited directly. When this repo is updated, every site repo picks up the changes on its next build.

### How a site build works

1. The site repo's workflow triggers (push, cron, or manual)
2. It calls the reusable workflow in this repo (`reusable.yml`)
3. The runner checks out the site repo, then checks out this template repo
4. Template files (scripts, Gemfile, config, selected theme) are copied into the site repo's workspace
5. The fetch script logs into Payload CMS with the site's credentials and fetches all accessible posts
6. The fetched posts and copied template files are committed back to the site repo as a content snapshot
7. Jekyll builds the site and deploys to GitHub Pages

The snapshot commit means each site repo accumulates a complete git history of its content and templates at every point in time — the site can be rebuilt at any historical state without the CMS being available.

---

## Themes

Themes live in `_themes/<name>/` and contain everything Jekyll needs to render the site:

```
_themes/
  default/              ← editorial theme (dark navy header, Cormorant Garamond)
    _layouts/
      default.html
      post.html
    index.html
  academic/             ← academic theme (white, top bar header, Lora serif)
    _layouts/
      default.html
      post.html
    index.html
```

To add a new theme, create a new directory under `_themes/` following the same structure. Site repos select a theme by name in their workflow file.

To update a theme, edit files under `_themes/` in this repo and push. All sites using that theme will pick up the changes on their next build.

---

## Creating a new site repo

### 1. Create the repository

Create a new empty repository on GitHub (public or private).

### 2. Add repository secrets

In the new repo: **Settings → Secrets and variables → Actions → New repository secret**

| Secret | Value |
|---|---|
| `CMS_URL` | The Payload CMS base URL, e.g. `https://your-cms.example.com` |
| `CMS_EMAIL` | Email of the Payload user whose access scopes the site content |
| `CMS_PASSWORD` | Password for that user |

The CMS user's access permissions determine which posts appear on the site. A user with access to all posts builds a complete site; a user scoped to a group or subject builds a filtered site.

### 3. Enable GitHub Pages

**Settings → Pages → Source → GitHub Actions**

### 4. Add the workflow file

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 * * * *'   # hourly; adjust as needed
  workflow_dispatch:

permissions:
  contents: write
  pages: write
  id-token: write

jobs:
  build:
    uses: jeffreycwitt/loyola-jekyll-cms-template/.github/workflows/reusable.yml@main
    with:
      theme: default        # or: academic
      group: ''             # optional: set to a group name (e.g. FA26_FR365) to filter posts
    secrets:
      CMS_URL:      ${{ secrets.CMS_URL }}
      CMS_EMAIL:    ${{ secrets.CMS_EMAIL }}
      CMS_PASSWORD: ${{ secrets.CMS_PASSWORD }}
```

### 5. (Optional) Override site title and description

Create `_config.site.yml` in the repo root:

```yaml
title: My Site Title
description: A short description
```

This merges with the template's `_config.yml` at build time. Use it for any per-site Jekyll configuration overrides.

### 6. Trigger the first build

Go to **Actions → Deploy → Run workflow**. The site will be live at `https://<org>.github.io/<repo-name>` once the run completes.

---

## Local development (template repo only)

Install dependencies:

```bash
bundle install   # Jekyll and Ruby gems
```

Copy `.env.example` to `.env` and fill in credentials:

```bash
cp .env.example .env
```

Fetch posts from the CMS and apply a theme, then serve:

```bash
npm run fetch
cp -r _themes/default/_layouts . && cp _themes/default/index.html .
bundle exec jekyll serve
```

To preview a different theme:

```bash
cp -r _themes/academic/_layouts . && cp _themes/academic/index.html .
bundle exec jekyll serve
```

> **Note:** The applied `_layouts/` and `index.html` at the repo root are working copies for local development. They are also committed as part of the snapshot in the standalone workflow. Do not edit them directly — edit the source files under `_themes/`.

---

## Workflows

### `fetch-and-deploy.yml` — standalone

Runs on this template repo itself. Triggers on push to `main`, hourly cron, and manual dispatch. Always uses the `default` theme. Useful for previewing theme changes before they propagate to site repos.

### `reusable.yml` — reusable

Called by site repos. Accepts a `theme` input (default: `default`) and three secrets (`CMS_URL`, `CMS_EMAIL`, `CMS_PASSWORD`). Contains all build logic so site repos need no build code of their own.

---

## Cross-organization use

To allow a site repo under a different GitHub organization to call the reusable workflow:

1. In this repo: **Settings → Actions → General → Access → Accessible from repositories in other organizations and enterprises**
2. In the site repo's `deploy.yml`: pass secrets explicitly (already the case in the template above — no change needed)

---

## Fetch script

`scripts/fetch-posts.js` — Node.js 18+, no external dependencies.

- Reads `CMS_URL`, `CMS_EMAIL`, `CMS_PASSWORD` from environment or `.env` file
- If credentials are provided, authenticates via `POST /api/users/login` and attaches a JWT to all requests
- If `CMS_GROUP` is set, filters posts to that group name (`where[group.name][equals]=<value>`); otherwise fetches all accessible posts
- Fetches all posts with pagination (`/api/posts?page=N&limit=100`)
- Fetches all users to resolve author names (`/api/users`)
- Converts Payload's Lexical rich-text JSON to HTML (paragraphs, headings, bold/italic/code, lists, links, images, YouTube video embeds)
- Writes one `_posts/YYYY-MM-DD-<slug>.html` file per post with Jekyll front matter
- Clears `_posts/` before each run so removed or inaccessible posts do not persist
