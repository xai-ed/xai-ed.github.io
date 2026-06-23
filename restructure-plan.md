# Plan: Reorganize XAI-ED web presence into hub + per-workshop subpaths

## Context

Today the GitHub org `xai-ed` has the workshop site living **in the org-pages repo itself** (`xai-ed/xai-ed.github.io`), serving the LAK26 workshop at the root of the custom domain `https://www.xai-ed.net/`. A second repo (`xai-ed/AXLE`, Festival of Learning 2026) exists separately.

Goal: make `www.xai-ed.net` a **community hub** (an EDM-style landing page for the XAI-ED community), and host each workshop as its own repo behind a path:
`www.xai-ed.net/LAK26`, `www.xai-ed.net/AXLE`, etc.

This maps cleanly onto **GitHub Pages org mechanics** — no DNS work, no custom routing layer:

- The `xai-ed.github.io` repo is the **org/user Pages site** → serves the **root** of the custom domain.
- Any **other repo** in the org is a **project Pages site** → automatically served at `www.xai-ed.net/<repo-name>/` once Pages is enabled, because the custom domain is verified on the org repo. Project repos must **not** carry their own `CNAME`.

So the slash routing the user wants is free. The real work is: (1) repo topology + Pages settings, and (2) making each Hugo workshop site **subpath-correct** (baseURL + link fixes), because the current content has hardcoded root-relative links that break under a `/LAK26/` prefix.

Decisions (confirmed with user):
- **Deliverable now: plan only** (this runbook). No code changes yet.
- **LAK26 = new repo**, copy content into it (fresh history). The existing `xai-ed.github.io` repo is **repurposed** as the hub (keeps the custom domain).
- **Hub: Hugo**, template TBD (may pick a new one).
- **Links: convert to Hugo baseURL-aware** (relref / relURL), not `canonifyURLs`.

---

## Target topology

| Repo | Pages type | Serves at | baseURL in hugo.toml |
|---|---|---|---|
| `xai-ed/xai-ed.github.io` | org page (root) | `https://www.xai-ed.net/` | `https://www.xai-ed.net/` |
| `xai-ed/LAK26` (new) | project page | `https://www.xai-ed.net/LAK26/` | `https://www.xai-ed.net/LAK26/` |
| `xai-ed/AXLE` (exists) | project page | `https://www.xai-ed.net/AXLE/` | `https://www.xai-ed.net/AXLE/` |
| future workshops | project page | `…/<repo>/` | `…/<repo>/` |

Custom domain (`www.xai-ed.net`) stays configured on the **org repo only**. DNS unchanged.

---

## Workstream A — Stand up `LAK26` (the current workshop site at a subpath) — ✅ DONE

Source = the current repo content. Steps:

1. **Create repo** `xai-ed/LAK26`. Copy the entire current site into it (everything except `.git`, `public/`, `resources/`, `node_modules/`).

2. **Set the subpath baseURL** in `hugo.toml:3`:
   ```toml
   baseURL = "https://www.xai-ed.net/LAK26/"
   ```
   Hugo prefixes every `relURL`/`absURL`/`relLangURL`/`RelPermalink`/menu link with the `/LAK26/` path automatically — so the **theme/layouts are already safe** (they use these funcs, audited: no hardcoded absolute paths in `layouts/`).

3. **Convert hardcoded content links** (the breakage). These resolve to the domain root and drop `/LAK26/` unless fixed. Known offenders (re-run the audit grep below to catch all):
   - Internal **page links** → `relref` shortcode (also fails the build on broken refs — free safety net):
     - `content/english/_index.md:19`: `href="/schedule/#keynote-details"` → `href="{{</* relref "schedule.md" */>}}#keynote-details"`
   - **Static assets** (images, PDFs in `static/images`, `static/papers`) → these are raw `<img src="/…">` / markdown `[paper](/papers/…)`. Raw HTML in markdown does **not** run templates, so add a tiny shortcode and route assets through `relURL`:
     - New `layouts/shortcodes/asset.html`:
       ```go-html-template
       {{- (.Get 0 | relURL) -}}
       ```
     - `content/english/schedule.md:27` and `content/english/_index.md:12`: `<img src="/images/benjamin_paassen.jpg" …>` → `src="{{%/* asset "/images/benjamin_paassen.jpg" */%}}"` (or convert to a Markdown image so Hugo render-image hooks apply).
     - `content/english/proceedings.md` lines 10–24: the 8 `[paper](/papers/XAI-Ed_2026_paper_*.pdf)` links → `[paper](/LAK26/papers/…)` won't survive a future rename, so prefer a render hook or the `asset` shortcode pattern. Simplest robust option for PDFs: a Markdown **render-link hook** (`layouts/_default/_markup/render-link.html`) that pipes root-relative destinations through `relURL`.

   Audit command (run in the LAK26 repo before/after) — finds every root-relative link in content:
   ```bash
   grep -rnE 'href="/|src="/|\]\(/' content/ | grep -vE 'https?://'
   ```
   Also check `config/_default/menus.en.toml` for any leading-slash `url =` (those go through `relLangURL` in the header partial, so they are safe, but verify).

4. **Strip platform configs not used by GitHub Pages** (optional cleanup, separate commit): `_redirects`, `netlify.toml`, `vercel.json`, `vercel-build.sh`, `amplify.yml`, `.gitlab-ci.yml` are dead weight here. The active deploy is `.github/workflows/main.yml` (push to `main` → Hugo build → Pages artifact). It needs **no baseURL override** — it reads `hugo.toml`. Keep it as-is.

5. **No `CNAME` file** in this repo (there isn't one today — good). The domain is inherited from the org repo.

6. **Enable Pages**: repo Settings → Pages → Source = **GitHub Actions**. First push to `main` deploys to `www.xai-ed.net/LAK26/`.

---

## Workstream B — Repurpose `xai-ed.github.io` as the community hub

1. Decide template (open: reuse Hugoplate for visual consistency, or pick a new Hugo theme). Build a fresh hub site (landing, about the community, list of workshops linking to `/LAK26`, `/AXLE`, …, past editions, CFP/news).
2. `baseURL = "https://www.xai-ed.net/"`.
3. Keep the custom domain in this repo's Settings → Pages (the verified domain stays here). Keep Source = GitHub Actions.
4. Replacing the old root content means old deep links like `www.xai-ed.net/schedule/` 404. **Optional**: add a render-link/redirect or a few stub pages redirecting old root paths → `/LAK26/…`. Low priority — LAK26 is an upcoming edition, links not yet widely distributed.

---

## Workstream C — Bring `AXLE` into the scheme

Same recipe as Workstream A applied to the existing `xai-ed/AXLE` repo:
- `baseURL = "https://www.xai-ed.net/AXLE/"`, convert any hardcoded content links, ensure **no own CNAME**, Pages Source = GitHub Actions.
- (Inspect AXLE separately — it's not in this workspace; it may or may not be Hugo/Hugoplate.)

---

## Sequencing (minimize downtime)

1. ~~Build + verify `LAK26` repo locally first (Workstream A) → enable its Pages → confirm `www.xai-ed.net/LAK26/` works **while root still shows the old site**.~~ ✅ DONE
2. Then repurpose the root repo to the hub (Workstream B). Root flips to the hub; LAK26 already live at its path.
3. AXLE (Workstream C) anytime — independent.

---

## Verification

For each workshop site (run in its repo):

```bash
# Build with the production subpath baseURL
hugo --gc --minify --baseURL "https://www.xai-ed.net/LAK26/"

# 1. No link leaks to the domain root (should print NOTHING):
grep -rnoE 'href="/[a-zA-Z]' public/ | grep -vE 'href="/LAK26/' | grep -vE 'https?:'
grep -rnoE 'src="/[a-zA-Z]'  public/ | grep -vE 'src="/LAK26/'  | grep -vE 'https?:'

# 2. Spot-check that pages/assets got the prefix:
grep -rn '/LAK26/papers/'   public/proceedings/index.html
grep -rn '/LAK26/schedule/' public/index.html
```

- **Local preview of the subpath**: `hugo server --baseURL "http://localhost:1313/LAK26/" --appendPort=false` then open `http://localhost:1313/LAK26/` and click through nav, keynote image, schedule anchor, and each proceedings PDF.
- **Post-deploy**: load `https://www.xai-ed.net/LAK26/`, verify nav, the Paaßen portrait image, the `#keynote-details` anchor, and one PDF download all resolve (no 404, no drop to root).
- **Hub**: `https://www.xai-ed.net/` shows the hub and its links to `/LAK26/` and `/AXLE/` work.

---

## Open items to confirm before/while executing

- Hub template choice (Hugoplate vs new).
- Canonical host: keep `www.xai-ed.net` (current baseURL) vs apex `xai-ed.net` — keep `www` unless a redirect is preferred.
- Whether old root deep-links need redirects (Workstream B step 4).
- AXLE's actual stack (confirm when that repo is in hand).
