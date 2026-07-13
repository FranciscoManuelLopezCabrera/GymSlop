# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

PacoGym (Beta) — a gym-routine tracker PWA. Vue 3, single `index.html`, **no build step, no package.json, no external CDN**. Deployable to any static host. UI text and domain terms are in Spanish.

## Commands

There is no build, lint, or test tooling. To develop:

- **Run locally:** serve the folder over HTTP (a `file://` open breaks the service worker and fetch). E.g. `python -m http.server 8000` then open `http://localhost:8000/`.
- **Deploy:** GitHub Pages from `main` / root (`.nojekyll` serves files verbatim), or drag the folder to Netlify Drop.
- **After any deploy that changes app assets, bump `CACHE` in `sw.js`** (`gymslop-v12` → `v13`). The service worker is cache-first; without a version bump, installed clients keep serving the old build until the cache name changes.

## Architecture

Everything runs from `index.html`: `<style>` (lines ~13–274), the app `<template>` markup (~277–615), and one inline `<script>` (~617–1221). `vue.global.prod.js` is Vue 3 vendored locally (offline, no CDN). `sw.js` + `manifest.json` + `icon.svg` make it an installable offline PWA.

### State (single source of truth)

One `createApp({...})` Options-API instance (`data`/`computed`/`methods`/`watch`). All persistent state lives in one `localStorage` key **`gymslop.v4`** with shape:

```
{ active: <profileName>, profiles: { <name>: Profile } }
```

`Profile = { type, days[], sets{}, swaps{}, log{}, week{} }`:
- `days[]` — `{ id, label, sub, ex[] }`; `ex = { n, s, r, w, backups?[], cardio? }` (name, sets, reps, weight).
- `sets{}` — keyed by **`"<dayId>.<exerciseIndex>"`** (`skey(i)`), value = array of per-set booleans (checked sets).
- `swaps{}` — same key → index into that exercise's `backups[]` (hot-swap when a machine is busy).
- `log{}` — ISO date (`YYYY-MM-DD`) → dayId trained, or `"rest"`.
- `week{}` — Monday-first weekday `0..6` → dayId; the weekly template used to bulk-fill the calendar (`applyWeek`).

Persistence is automatic: `watch` on `active` and deep-`profiles` calls `persist()`. Changing the persisted shape requires a migration.

### Migrations

`loadState()` reads `gymslop.v4`, else migrates from `gymslop.v3` (fixed A/B/C `routine` object → day-array via `fullDaysFrom`), else `gymslop.v2` (single profile + weights map). With nothing found it seeds one default profile `"Paco"` (Full Body). Keep this chain intact when touching state.

### Routines & exercise catalog (module-level consts, above `createApp`)

- `TEMPLATES` — the four routine types keyed `full` / `ppl` / `clases` / `custom`; `makeProfile(type)` deep-clones its `days`. `TYPE_OPTS` drives the create-profile UI.
- `EXERCISES` — the 160+ picklist catalog, each `{ n, g }` (name, muscle group); rendered in `GROUP_ORDER`. Search is accent-insensitive via `norm()` (NFD + strip diacritics), matching name **or** group.

### Views & UX mechanics

- Three views via `this.view`: `train`, `cal` (calendar), plus the routine **editor**; a profile picker/creator gates entry when `needsPick`.
- **Rest timer:** starts automatically from `toggleSet` (marking a set) → `startRest`. Uses Web Audio (`beep`, square-wave 6-tone ~3s), `navigator.vibrate`, and a screen `wakeLock` re-acquired on `visibilitychange`.
- **PWA install:** `beforeinstallprompt` is captured into `installEvent`; `installApp` falls back to manual per-browser steps (iOS Safari / Firefox / Samsung).

### Gotcha: keep native/browser objects out of Vue's reactive data

`installEvent`, `audioCtx`, and `wakeLock` are module-level `let`s **on purpose** — wrapping them in Vue's reactive proxy breaks the native APIs. Do not move them into `data()`.
