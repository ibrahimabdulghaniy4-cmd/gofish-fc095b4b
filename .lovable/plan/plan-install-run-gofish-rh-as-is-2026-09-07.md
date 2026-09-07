# Plan: Install & run gofish_rh as-is

## Goal
Replace the current project with the contents of https://github.com/lockhoodfun-sys/gofish_rh.git ("Pulau Pancing" — a 3D fishing game built on TanStack Start, same stack as this project), install its dependencies, and run it unchanged, using the `.env` file that already exists in the repo.

## Steps
1. Copy the cloned repo files over the project root (`/dev-server`), excluding `.git`. This replaces the template placeholder files (src/, package.json, vite config, public/, supabase/, drizzle/, etc.) with the repo's own versions.
2. Keep the repo's `.env` exactly as-is (Supabase URL + keys, admin wallet, test-coins flag) — no values changed or displayed.
3. Install dependencies with `bun install` (the repo ships a `bun.lock`).
4. Verify: let the dev server reload, check the build log for errors, and open the preview to confirm the game renders at `/`. Fix only what's strictly needed to make it run; otherwise no code changes ("apa adanya").

## Technical notes
- The repo is a TanStack Start + React Three Fiber app, same framework as this project, so it drops in cleanly.
- Repo assets (3D models in `public/models/`, `public/draco/`) are included, so it runs offline/standalone.
- It uses Supabase via the provided env keys; no Lovable Cloud setup is needed or will be added.
