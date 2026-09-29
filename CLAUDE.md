# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

생활계획표 — Daily Dial: a single-page PWA (circular daily planner). No build step, no package manager, no framework — vanilla HTML/CSS/JS in one file plus a manifest and service worker.

## Files

- [index.html](index.html) — the entire app (styles, markup, and logic in one file). This is where almost all work happens.
- [sw.js](sw.js) — offline cache service worker. Bump `CACHE` (`daily-dial-v1`) whenever cached asset contents change, or returning users will keep the stale version.
- [manifest.json](manifest.json) — PWA install metadata.
- [README.md](README.md) — Korean end-user documentation (features, usage, deployment). Update this when user-facing behavior changes.

## Running / testing

There is no build, lint, or test command — this is intentionally dependency-free.

- Open `index.html` directly in a browser to iterate (double-click, or any static file server).
- PWA install and offline caching only activate when served over HTTPS (or via a local server); opening the file directly (`file://`) skips service worker registration (guarded in [index.html:717](index.html:717) by checking `location.protocol.startsWith("http")`).
- After changing cached assets, bump the `CACHE` constant in [sw.js:1](sw.js:1) so clients pick up the update.
- Manual test checklist after any change: add/edit/delete a task by dragging on the dial, switch AM/PM (`btnHalf`) and 모아보기/나눠보기 (`btnView`), save/open/duplicate/delete in 보관함 (archive), verify PNG export and print view.

## Architecture

Everything lives in the `<script>` block of [index.html](index.html), organized top-to-bottom as (search for the `/* ===== ... ===== */` section headers):

1. **Storage** — `store` wraps `localStorage` with an in-memory fallback if storage access throws.
2. **State** — a single `plan` object is the source of truth: `{ id, name, snap, tasks: [{id,s,e,label,color,star}], notes }`. Times (`s`/`e`) are floats in hours (0–24). `mode` (`"am" | "pm" | "all"`) controls which half of the dial is currently rendered — it's view-only and never changes task data.
2b. Two localStorage keys: `dd:current` (the in-progress draft, autosaved on every mutation via `persist()`) and `dd:plans` (the archive array, written only by explicit Save/duplicate/delete actions in 보관함).
3. **Geometry** — pure functions mapping time ↔ SVG angle/coordinates (`timeToDeg`, `pt`, `sectorPath`), parameterized by `modeRange()` so the same rendering code serves AM, PM, and full-24h views.
4. **Dial rendering** (`renderDial`) — clears and redraws the SVG from scratch each time: task sectors, hour ticks, labels/stars, live drag preview, and the current-time needle. There is no diffing; any state change re-renders the whole SVG.
5. **Pointer interaction** — raw `pointerdown/move/up` handlers on the SVG implement drag-to-create: accumulated angle deltas convert to hours, snapped to `plan.snap` (15 or 30 min). In 모아보기 (`mode==="all"`) a drag can wrap past midnight, producing two intervals (`intervalsFromDrag` returns 1 or 2 `[s,e]` pairs).
6. **Task mutation** (`addIntervals`) — inserting a new/edited interval **splits or removes** any existing tasks that overlap it (interval subtraction), rather than allowing overlapping sectors. This is the core invariant: tasks on the dial never overlap.
7. **Edit dialog / task list / notes / archive / PNG export** — each a self-contained block wiring up its own DOM elements and event listeners; all follow the same style (mutate `plan` → `persist()` → `renderAll()`).

Key conventions to preserve when editing:
- All rendering is regenerate-from-state; don't introduce incremental DOM patching for the dial.
- Time values are hours-as-floats, not minutes or Date objects — keep new code consistent with `fmt()`/`fmtDur()` for display.
- `mode` only affects what's drawn/edited in the current view, never the underlying stored task times (which are always absolute 0–24 hour values).
- Every state mutation must call `persist()` (writes `dd:current`) and then `renderAll()` (or the specific render function) to keep the DOM in sync — there's no reactive binding.
- Colors are restricted to the 8-value `PALETTE` array for readability; don't allow arbitrary colors in the editor.
