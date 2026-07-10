# Email Marketing — Team Documentation

Shareable architecture guide for the Email Marketing Agent.

## Quick open (local)

```bash
open docs/email-marketing/index.html
```

Or double-click `index.html` in Finder.

## Publish for the team

Your site is **static HTML** — no build step. Pick one:

### GitHub Pages (recommended — free, stays in GitHub)

**Easiest if this repo is on GitHub.** Your files live in a subfolder (`docs/email-marketing/`), so use a small Actions workflow (one-time setup):

1. **Push** the repo to GitHub (if not already).
2. **Repo → Settings → Pages**
   - **Build and deployment → Source:** `GitHub Actions`
3. **Add** `.github/workflows/email-marketing-docs.yml` (see below), commit, push to `main`.
4. After the workflow runs, your site is at:
   `https://<org-or-user>.github.io/<repo-name>/`

Workflow file to add:

```yaml
name: Email marketing docs

on:
  push:
    branches: [main]
    paths:
      - 'docs/email-marketing/**'
      - '.github/workflows/email-marketing-docs.yml'
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: email-marketing-docs
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deploy.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: docs/email-marketing
      - id: deploy
        uses: actions/deploy-pages@v4
```

No `npm install`, no build command — it uploads the folder as-is.

---

### Vercel (also easy — good if you already use Vercel)

1. Go to [vercel.com](https://vercel.com) → **Add New Project** → import this GitHub repo.
2. **Root Directory:** `docs/email-marketing`
3. **Framework Preset:** Other
4. **Build Command:** leave empty
5. **Output Directory:** `.` (or leave default)
6. Deploy → you get a URL like `https://your-project.vercel.app`

Redeploys automatically on every push to `main`.

---

### Which to pick?

| | GitHub Pages | Vercel |
|---|-------------|--------|
| **Setup** | Enable Pages + one workflow file | Connect repo, set root folder |
| **Cost** | Free on public repos | Free tier |
| **URL** | `github.io/...` | `vercel.app/...` |
| **Best for** | Docs tied to the repo | Custom domain / team already on Vercel |

**For a team doc inside this repo, GitHub Pages is usually the simplest.**

---

### Option C — Local preview only

```bash
cd docs/email-marketing
npx serve .
# Open http://localhost:3000
```

## Documents

| File | Format |
|------|--------|
| [`index.html`](index.html) | White-theme HTML guide with sidebar + DB sections |
| [`architecture-guide.md`](architecture-guide.md) | Markdown (same content) |
| [`../email-marketing-implementation-overview.md`](../email-marketing-implementation-overview.md) | Technical spec |
| [`../email-marketing-db-schema.sql`](../email-marketing-db-schema.sql) | SQL DDL |

## Sections in the HTML guide

1. System overview  
2. Two services (marketing-svc vs deepagent)  
3. **Step 1** — Onboarding entry point (hotels + users)  
4. **Step 2** — Onboarding flow (scrape, KB, assets)  
5. **Step 3** — Login & dashboard  
6. **Step 4** — Layout templates  
7. **Step 5** — Campaign creation  
8. **Step 6** — Preview & editor  
9. **Step 7** — Recipients  
10. **Step 8** — Send email  
11. **Step 9** — Scheduling  
12. **Step 10** — Analytics  
13. Database summary  
