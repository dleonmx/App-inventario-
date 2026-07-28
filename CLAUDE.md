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

## Supabase configuration

`SUPABASE_URL` and `SUPABASE_ANON_KEY` are hardcoded near the top of the script (index.html:258-259). These are the public anon key and project URL, safe to expose client-side per Supabase's design — do not treat them as secrets, but also don't casually replace them without confirming with the user which Supabase project should back the app.

The only expected table is `inventario_kv` with `key` (text, primary/unique) and `value` (text, JSON-encoded) columns.

## Language

All user-facing copy is in Spanish (`lang="es"`). Keep new UI strings consistent with the existing Spanish tone (informal "tú" address, warm/family-oriented phrasing).
