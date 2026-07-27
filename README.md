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
    index.html          ← portfolio-aware; falls back to date-sorted post list
    portfolios.html     ← multi-portfolio browser at /portfolios/
    tags.html           ← tag index at /tags/
  academic/             ← academic theme (white, top bar header, Lora serif)
    _layouts/
      default.html
      post.html
    index.html
    portfolios.html
    tags.html
  FA26_FR365/           ← Les Misérables class theme (parchment/crimson, EB Garamond)
    _layouts/
      default.html
      post.html
    index.html          ← welcome section in regular mode; portfolio-aware
    portfolios.html
    tags.html
    about.html          ← hardcoded course and book description
```

To add a new theme, create a new directory under `_themes/` following the same structure. Any `.html` files at the theme root are automatically copied and snapshotted — add an `about.html` or other static pages freely.

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
      theme: default        # or: academic, FA26_FR365
      group: ''             # optional: filter posts to a Payload group name
      portfolio: ''         # optional: see Portfolio modes below
    secrets:
      CMS_URL:      ${{ secrets.CMS_URL }}
      CMS_EMAIL:    ${{ secrets.CMS_EMAIL }}
      CMS_PASSWORD: ${{ secrets.CMS_PASSWORD }}
```

### 5. (Optional) Enable portfolio mode

The `portfolio` input controls whether the site is built from a Payload Portfolio collection instead of the full post feed.

| `portfolio` value | Behavior |
|---|---|
| _(not set)_ | Regular mode: all accessible published posts, sorted by date |
| A portfolio title or numeric ID | Single-portfolio mode |
| `all` | Multi-portfolio mode |

**Single-portfolio mode** — pass the portfolio's title (or its numeric Payload ID) to build the site from exactly one portfolio's post list in the curator-defined order:

```yaml
with:
  theme: FA26_FR365
  group: FA26_FR365
  portfolio: "Student Selections"
```

The index page shows a portfolio header (title + description) followed by posts in the portfolio's order. The `/portfolios/` page shows the same list as a secondary URL. A `_data/portfolio.yml` file is written with the ordered post IDs so Liquid can look them up by `cms_id` rather than relying on Jekyll's date sort.

**Multi-portfolio mode** — pass `all` to aggregate every portfolio accessible to the CMS user (filtered by `group` if set):

```yaml
with:
  theme: FA26_FR365
  group: FA26_FR365
  portfolio: all
```

The index page shows all posts date-sorted (same as regular mode); the `/portfolios/` page shows each portfolio as a section with its posts in the curator-defined order. All unique posts across portfolios are written to `_posts/`; duplicate posts (the same post in multiple portfolios) are written once. A `_data/portfolios.yml` file is written with the full ordered list for each portfolio.

### 6. (Optional) Override site title and description

Create `_config.site.yml` in the repo root:

```yaml
title: My Site Title
description: A short description
```

This merges with the template's `_config.yml` at build time. Use it for any per-site Jekyll configuration overrides.

### 7. Trigger the first build

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
cp -r _themes/default/. .
bundle exec jekyll serve
```

To use portfolio mode locally, set `CMS_PORTFOLIO` before fetching:

```bash
CMS_PORTFOLIO="Student Selections" npm run fetch   # single portfolio
CMS_PORTFOLIO=all CMS_GROUP=FA26_FR365 npm run fetch  # multi-portfolio
cp -r _themes/FA26_FR365/. .
bundle exec jekyll serve
```

To preview a different theme:

```bash
cp -r _themes/academic/. .
bundle exec jekyll serve
```

> **Note:** The applied `_layouts/`, `index.html`, and other theme files at the repo root are working copies for local development. Do not edit them directly — edit the source files under `_themes/`.

---

## Workflows

### `fetch-and-deploy.yml` — standalone

Runs on this template repo itself. Triggers on push to `main`, hourly cron, and manual dispatch. Always uses the `default` theme. Useful for previewing theme changes before they propagate to site repos.

### `reusable.yml` — reusable

Called by site repos. Accepts `theme` (default: `default`), `group`, and `portfolio` inputs plus three secrets (`CMS_URL`, `CMS_EMAIL`, `CMS_PASSWORD`). Contains all build logic so site repos need no build code of their own.

---

## Cross-organization use

To allow a site repo under a different GitHub organization to call the reusable workflow:

1. In this repo: **Settings → Actions → General → Access → Accessible from repositories in other organizations and enterprises**
2. In the site repo's `deploy.yml`: pass secrets explicitly (already the case in the template above — no change needed)

---

## Fetch script

`scripts/fetch-posts.js` — Node.js 18+, no external dependencies.

- Reads `CMS_URL`, `CMS_EMAIL`, `CMS_PASSWORD`, `CMS_GROUP`, `CMS_PORTFOLIO` from environment or `.env` file
- If credentials are provided, authenticates via `POST /api/users/login` and attaches a JWT to all requests
- Fetches all users to resolve author email addresses (`/api/users`)
- Converts Payload's Lexical rich-text JSON to HTML (paragraphs, headings, bold/italic/code, lists, links, images, YouTube video embeds)
- Writes one `_posts/YYYY-MM-DD-<slug>.html` file per post with Jekyll front matter (`title`, `date`, `cms_id`, `author`, `tags`)
- Clears `_posts/` and any stale `_data/portfolio*.yml` files before each run

**Regular mode** (no `CMS_PORTFOLIO`): fetches all published posts filtered by `CMS_GROUP` if set, sorted by date. No `_data/` files written.

**Single-portfolio mode** (`CMS_PORTFOLIO=<title or id>`): fetches the named portfolio at depth=2, writes its posts to `_posts/`, and writes `_data/portfolio.yml` with the ordered `cms_id` list. The theme's `index.html` reads `site.data.portfolio` to render posts in portfolio order rather than date order.

**Multi-portfolio mode** (`CMS_PORTFOLIO=all`): fetches all portfolios (filtered by `CMS_GROUP` if set) at depth=2, deduplicates posts across portfolios, writes all posts to `_posts/`, and writes `_data/portfolios.yml` with an entry per portfolio containing its ordered `cms_id` list. The theme's `index.html` and `/portfolios/` page both read `site.data.portfolios`.
