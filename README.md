# TILT

Read-only risk & discipline tool for Hyperliquid perp traders. This repo holds the **web dashboard** and **docs**. The detection engine (SQL + scheduled jobs + the data API) lives in Supabase.

## Files
- `index.html` — the dashboard (open it, or host it; see below).
- `docs/SYSTEM.md` — full system blueprint (architecture, tables, functions, how to change things).
- `supabase/dashboard.ts` — the edge-function data API (reference copy; deployed in Supabase).

## Access key
The dashboard asks for an access key the first time and remembers it in your browser. Current key: `grull-tilt-7f3a9c2e`. (No secret is stored in this file, so it's safe to host publicly.)

## Publish the dashboard with GitHub Pages (free, auto-deploys on every change)
1. Create a new repository on github.com (e.g. `tilt`). Public is fine — no secrets live here.
2. Upload `index.html` (and the `docs/` folder if you like) to the repo.
3. In the repo: **Settings → Pages → Build and deployment → Source: "Deploy from a branch" → Branch: `main` / root → Save.**
4. Wait ~1 minute. Your dashboard is live at `https://<your-username>.github.io/tilt/`.
5. Open it, enter the access key when prompted, and share the URL with the desk.

**To update it later:** edit `index.html` in GitHub (or have the assistant give you a new version), commit, and Pages re-publishes automatically within a minute.

## Switching wallets
The dashboard has a mainnet/testnet dropdown. Anyone you add to the engine's trader list appears there automatically.
