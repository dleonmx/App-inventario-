# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Inventario" — a single-file web app (`index.html`) for families to jointly inventory a household's belongings and let each family member mark which items they'd like to keep. It surfaces conflicts when more than one person claims the same item.

The entire application — markup, CSS, and JS — lives in `index.html`. There is no build step, no bundler, no package.json, and no test suite. Persistence is handled by Supabase (loaded via CDN script tag, `@supabase/supabase-js@2`).

## Running / developing

There is no build or dev server. Open `index.html` directly in a browser, or serve the directory with any static file server (e.g. `npx serve .`) to avoid `file://` restrictions on fetch calls to Supabase.

There are no lint or test commands configured in this repo.

## Architecture

Everything is inside one IIFE in the `<script>` tag at the bottom of `index.html`. Key parts:

- **Storage layer (`storeGet`/`storeSet`/`storeDelete`)**: a generic key-value store backed by a single Supabase table `inventario_kv` (columns: `key`, `value` — value is JSON-stringified). All app data — config, items, per-person selections, and each item's photo — is stored as separate KV rows. Photos are stored under a per-item key (`photoKey(id)`), as compressed base64 JPEG data URLs (downsized via an offscreen `<canvas>`, see `fileToCompressedDataUrl`) so they fit as text values.
- **App state**: a single `state` object (`route`, `person`, `config`, `items`, `selections`, `reviewIndex`) drives a hand-rolled render loop — no framework. `render()` rebuilds `root.innerHTML` from string-concatenated HTML based on `state.route`.
- **Routing**: client-side only, via `navigate(route, extra, opts)` and a manual `navHistory` stack for back navigation (`window.__back`). Routes: `loading`, `home`, `admin`, `review`, `summary`, `results`.
- **Screens** (each a function returning an HTML string): `homeScreen` (pick a family member), `adminScreen` (configure title/subtitle/family names, add/remove items with photos — this is the "organizer" view), `reviewScreen` (swipe-style one-item-at-a-time decision UI per person), `summaryScreen` (per-person results after finishing review, flags conflicts), `resultsScreen` (global view of who claimed what, across all people).
- **Event wiring**: no framework event binding — inline `onclick`/`onchange` attributes call functions exposed on `window` (e.g. `window.__nav`, `window.__decide`, `window.__addItem`, `window.__saveConfig`, `window.__deleteItem`, `window.__selectPerson`, `window.__onPhotoSelected`, `window.__back`).
- **Escaping**: all user-provided strings interpolated into HTML must go through `esc()` (sets `textContent` then reads `innerHTML` to escape) — follow this pattern for any new dynamic content to avoid XSS.
- **Optimistic UI**: decisions in `reviewScreen`/`window.__decide` advance the UI immediately and persist to Supabase in the background, surfacing a toast (`showToast`) on save failure rather than blocking navigation.

## Concurrency rules — read before touching any save path

Two people using the app at once used to lose each other's work: every write sent the snapshot the page downloaded on load, so the last person to press a button silently overwrote everything the others had done since. Fixed in `d726248`. Keep these rules:

- **Never write a whole row built from `state`.** Use `updateRow(key, fallback, mutate)`: it re-reads the row from the server, applies just the delta, and saves — queued per key so two quick taps can't read the same stale value. Used by `__decide`, `__setWinner`, `__toggleDonePersona`, `__addItem`, `__deleteItem`.
- **`storeGet` returns the fallback when the network fails.** Never use it to read a row you are about to rewrite — you would save an empty row over good data. Use `storeGetRaw`/`storeSetRaw`, which throw.
- **Selections are one row per person** (`inventario:sel:<Nombre>`, see `selKey()`), never one shared row. The old shared `inventario:selecciones` row is split into per-person rows on the first `loadAll()` and then deleted, only once every new row saved. `loadSelections()` reads them all in a single prefix query.
- **Toggles send the resulting value, not "toggle again".** The local state decides the new value; the server write applies that value. Otherwise two people toggling different things undo each other.

To verify a concurrency change: copy `index.html` to a scratch dir replacing `'inventario:` with `'ZZTEST:`, serve it (`python -m http.server`), open it in two tabs, act in both, then read the rows back. Never test against the real `inventario:*` keys.

## Current state and what's pending

Working and verified: per-person rows, re-read before saving, conflict resolution, PDF export, full inventory wipe.

Pending, in order of impact:

1. **Photos are all downloaded on startup** (`loadAll()`), with no cache. Measured with 300 items: ~21 MB and 303 HTTP requests per page load (~35 s on 4G), which also burns through Supabase's free 5 GB/month in weeks. Fix: load a photo only when it's actually shown. This is the one that raises the item ceiling from ~100 to thousands.
2. **The app is single-tenant.** There is one inventory in one Supabase project, so any second household would share the same data. A per-client namespace picked from the URL (`?cliente=...` → `inventario:e-<slug>:...`, with no parameter keeping today's keys untouched) was designed but **not implemented**.
3. **No auth.** Anyone with the link can add, edit, or wipe the whole inventory. Supabase RLS is the fix. This is the blocker before showing it to anyone outside the family.

Note: `inventario:config` currently lists "Pedro" twice in `familyNames`. Two people with the same name share one selections row — same behaviour as before the fix, not a regression, but worth renaming if they are different people.

## Supabase configuration

`SUPABASE_URL` and `SUPABASE_ANON_KEY` are hardcoded near the top of the script (index.html:258-259). These are the public anon key and project URL, safe to expose client-side per Supabase's design — do not treat them as secrets, but also don't casually replace them without confirming with the user which Supabase project should back the app.

The only expected table is `inventario_kv` with `key` (text, primary/unique) and `value` (text, JSON-encoded) columns.

## Language

All user-facing copy is in Spanish (`lang="es"`). Keep new UI strings consistent with the existing Spanish tone (informal "tú" address, warm/family-oriented phrasing).
