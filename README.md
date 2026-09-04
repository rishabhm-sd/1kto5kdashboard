# HITS Retention Dashboards

A small static site with three tabs, each one of the HITS/Revenue retention dashboards:

- `index.html` — the tab shell (open this one)
- `retention.html` — HITS Churn-Cohort Retention
- `retention-3k.html` — 1-5K 3K Retention
- `timing.html` — HITS Churn Timing Diagnostic

Each dashboard is fully self-contained (no build step, no external data calls) and also works fine opened on its own — `index.html` just loads the other three in an iframe per tab and keeps each one's filters/toggle state while you switch away and back.

## Host it on GitHub Pages (free, ~2 minutes)

1. Create a new GitHub repository (public, or private if your plan supports Pages on private repos) — e.g. `hits-retention-dashboards`.
2. Add these four files to the repo root (drag-and-drop on github.com works, or via git):
   ```
   git init
   git add index.html retention.html retention-3k.html timing.html README.md
   git commit -m "Add HITS retention dashboards"
   git branch -M main
   git remote add origin https://github.com/<your-username>/<your-repo>.git
   git push -u origin main
   ```
3. In the repo, go to **Settings → Pages**.
4. Under "Build and deployment", set **Source** to "Deploy from a branch", pick branch **main** and folder **/ (root)**, then Save.
5. GitHub will publish it at `https://<your-username>.github.io/<your-repo>/` within a minute or two — refresh the Pages settings page to see the live link once it's ready.

That's it — no server, no build step. Since GitHub Pages serves everything from one origin, the tab-switching and auto-sizing between the three dashboards works exactly as it does locally.

## Updating the data later

Each dashboard's numbers are inline in its own `<script>` block — there's no live database connection from the page itself. A scheduled task refreshes them automatically every Monday at 1pm IST, straight from Metabase, and updates this folder's copies as well as the three published artifact pages. See `DATA_REFRESH_RUNBOOK.md` in this folder for exactly what it pulls and how it recomputes each constant. GitHub push isn't wired into that task yet — once you've set up this repo with git credentials, tell Claude the local clone path and it'll add a `git add/commit/push` step so the live site updates automatically too. Until then, after each refresh you'll want to pull the latest `retention.html` / `retention-3k.html` / `timing.html` from this folder and push them yourself.

## Running it locally first (optional)

Iframes need a real HTTP origin to talk to each other's height — opening `index.html` directly via `file://` mostly works in Chrome/Edge but can be blocked in some browsers. To preview exactly like GitHub Pages will serve it:

```
python3 -m http.server 8000
```

then open `http://localhost:8000/index.html`.
