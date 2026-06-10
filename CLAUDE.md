# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

TimePlanner.AI — a time-management / planning app that is a **single static file**: `index.html`
(~6,400 lines). It contains the CSS (in `<head><style>`), the markup, and all logic in one
trailing IIFE (`<script>` at ~line 1871). There is **no build step, no bundler, no package
manager, and no test suite**. The only external code is two CDN `<script>` tags loaded at runtime:
`@supabase/supabase-js` (cloud sync) and `sortablejs` (drag-and-drop).

## Develop / run / deploy

- **Run locally:** serve the directory over HTTP (don't `file://` it — Supabase OAuth redirect and
  localStorage origin need a real origin). `npx -y http-server -p 8000 -c-1` then open
  `http://localhost:8000`.
- **Deploy:** push to `main` on `github.com/colesr/timemgmt`; GitHub Pages serves the live build at
  `https://colesr.github.io/timemgmt/`. Pages can lag a minute or two before serving the new build.
- All editing happens directly in `index.html`. Keep CSS/markup/JS in that one file — do not split
  it out.

## Data model (read before touching state)

A single in-memory `state` object is the whole app's source of truth:
`{ portfolios, projects, activities, entries, panelLayouts }`.

The hierarchy is **portfolio → project → activity**, with `entries` being individual time-tracking
sessions. Links are by id: `project.portfolioId`, `activity.projectId`, `entry.activityId`
(`entry.end === null` means a timer is currently running — `getRunning()`).

- **The UI label "Task" maps to the internal key `activities` / `activityId`.** This rename was
  never carried into the data model on purpose — changing the keys breaks every persisted and
  synced state blob. Keep using `activities`/`activityId` in code.
- Cross-hierarchy **dependencies** are stored on each entity as `dependencies: [{kind, id}]` where
  `kind` is `"portfolio" | "project" | "activity"`; resolve them via `resolveDep()`.
- **Image attachments** are compressed data-URL strings stored inline in `entity.images[]` (so they
  ride along inside the synced state blob — there is no separate file storage).

## Persistence & migrations (the easy thing to get wrong)

- `save()` writes `state` to localStorage (`STORAGE_KEY = "timetrack.v2"`) **and** triggers a
  debounced `cloudPush()`. Theme is a separate key, `timeplanner.theme`.
- New fields are not guaranteed to exist on older saved data, so both `load()` (~line 1923) and
  `cloudPull()` (~line 6293) run **backfill loops** that default missing fields on every entity.
  **When you add a field to the model, add its backfill in BOTH places** — otherwise data from an
  older device/localStorage will be missing it after a sync pull.

## Cloud sync (Supabase)

- Config constants `SUPABASE_URL` / `SUPABASE_ANON_KEY` are at the top of the script (~line 1888);
  the SQL schema for the single `user_state` table (`user_id`, `state jsonb`, `updated_at`) plus its
  row-level-security policy is documented in the comment just above. Auth is GitHub OAuth.
- One row per user holds the entire `state` as JSON. `cloudPush()` is debounced (~800ms);
  `cloudPull()` runs on sign-in and, if local and cloud differ, prompts the user to pick which wins.
- Sync is optional: if the Supabase constants are left as `YOUR_...` placeholders, `cloudEnabled`
  is false and the app runs purely on localStorage.

## UI architecture

- **Routing** is hash-based. `parseHash()` turns `location.hash` into a `view` object
  (`home`, `reports`, `plan`, `portfolio`, `project`); `navigate()`/`viewHref()` go the other way; a
  `hashchange` listener re-renders.
- **Rendering** is full-redraw: `render()` builds a fresh `#view` subtree with one `renderXxxView()`
  per route, then `swap()`s it in with directional enter/leave CSS transitions (depth-aware
  forward/back animation). After any state mutation, call `save()` then `render()`.
- **Drag-and-drop** (panel reordering, hierarchy moves) uses SortableJS, wired per-view in
  `applyDragAndDrop()` / `initSortablesAfterAttach()`; ordering state persists in
  `state.panelLayouts`.

## The "AI" Guru

The planning assistant (`guruRespond()`, ~line 5560) is **fully local and rule-based** — it does
keyword/intent routing over regexes and answers from analyses grounded in the user's own `state`
(`diagnose()`, `suggestPriorities()`, `planWeek()`, `findRisks()`, `summarizeEntity()`, etc.). There
is no external LLM API call. If asked to make the Guru smarter, you're editing these
data-analysis functions, not adding a network dependency.
