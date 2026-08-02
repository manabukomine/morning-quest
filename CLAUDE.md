# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

朝クエスト (Morning Quest) is a Japanese-language installable PWA that turns a kid's morning routine into a checklist game: complete tasks (wake up, get dressed, eat breakfast, brush teeth, pack bag, leave the house) to earn points, build a daily streak, and level up. All UI text, task names, and in-app messages are in Japanese.

There is no build system, package manager, or framework. This is a static site: open `index.html` directly, or serve the four files over HTTP(S) for the service worker/PWA install to work.

## Files

- `index.html` — the entire application: CSS in a single `<style>` block, HTML markup, and vanilla JS in two `<script>` blocks. This is the only file you'll typically edit.
- `manifest.json` — PWA manifest (name, icons, theme colors, `standalone` display).
- `service-worker.js` — cache-first service worker; caches `index.html`, `manifest.json`, and the icons for offline use.
- `icons/` — `icon-192.png`, `icon-512.png` referenced by the manifest.
- `README.md` (Japanese) — deployment notes for GitHub Pages.

## Development workflow

There is no build, lint, or test tooling in this repo — edit `index.html` directly and verify in a browser.

- **Local preview:** open `index.html` directly in a browser, or run a static server from the repo root (e.g. `python3 -m http.server`) so the service worker and manifest load correctly (service worker registration is skipped when opened via `file://`).
- **Deployment:** GitHub Pages, "Deploy from a branch" on `main` / root. Per `README.md`, the files to publish are `index.html`, `manifest.json`, `service-worker.js`, `.nojekyll`, and `icons/icon-192.png` / `icons/icon-512.png`. Note: `.nojekyll` is referenced by the README but is not currently present in the repo — add it if Pages starts mangling the deploy.
- **Cache busting:** any time you change `index.html`, `manifest.json`, or the icons, bump `CACHE_NAME` in `service-worker.js` (currently `'morning-quest-v1'`). Without this, returning users keep getting the stale cached `index.html` from the service worker's cache-first `fetch` handler.

## Architecture (inside `index.html`)

**Config → State → Render loop**, all in plain ES5-style JS (`var`, function declarations, no modules/build step, no external JS deps — only a Google Fonts `@import` in CSS).

1. **Config data** — `TASKS` (array of `{id, name, icon, iconBg, time, pts, msgs[]}`) and `BAG_ITEMS` (array of `{id, name, icon}`). Adding/removing/reordering a morning task or bag-check item means editing these arrays; the rest of the UI derives from them.
2. **State** — a single `state` object persisted to `localStorage` under key `mqState_v2` (`STORAGE_KEY`), read/written via `loadState()` / `saveState()`. Shape: `{ version, done: {taskId: true}, bagPacked: {itemId: true}, totalPts, completedDates: {dateKey: true}, streak, lastDate, soundMuted, soundVolPct }`. `loadState()` falls back to the legacy `mqState` key and fills in defaults for any missing field, so it's tolerant of older saved shapes — preserve that tolerance when changing the schema.
   - A day-rollover check runs at load time: if `state.lastDate` isn't today, `done`/`bagPacked` are cleared but cumulative fields (`totalPts`, `streak`, `completedDates`) survive.
   - State is also flushed on `pagehide` and on `visibilitychange` → hidden, since mobile browsers can kill the page without firing `beforeunload`.
3. **Render functions** (`renderTasks`, `renderBag`, `renderProgress`) rebuild DOM from `state` + config using the `el(tag, cssText, text)` helper and a small `svgCheck()` helper for the checkmark icon — there's no diffing, they just clear `innerHTML` and rebuild. Call the relevant `render*()` after any state mutation.
4. **Interaction handlers** (`toggleTask`, `toggleBag`) mutate `state`, call `saveState()`, then re-render and trigger side effects (popup, sound, confetti). Completing the last task triggers the full-screen "complete" celebration and updates the streak (`completedDates` / `streak` logic lives in `showComplete()`).
5. **Timer** (`updateTimer`/`updateTimerWithSound`, driven by `setInterval(..., 1000)`) shows elapsed time since app launch (`APP_START`, an in-memory timestamp — not persisted) and the next incomplete task, and plays a warning tone (`playTimerWarning`) when the next task's scheduled `time` is within 3 minutes.
6. **Audio** is all synthesized via the Web Audio API (`playTone`, `playClear`, `playFanfare`, `playStartupJingle`, `playTimerWarning`, and a procedurally looped `startBGM`/`stopBGM`) — there are no audio asset files. Every sound function checks `state.soundMuted` first. `AudioContext` is created lazily and resumed on the first user click (browsers block autoplay before a user gesture).
7. **Backup / restore** is manual, JSON-based, and localStorage-only (no backend/sync):
   - `exportData()` serializes `state` and both shows a copy-to-clipboard modal (`showBackupModal`, needed for iOS Safari where downloads don't work well) and attempts a `data:` URI file download for desktop/Android.
   - `importData()` reads a user-selected JSON file, merges it over the default state shape, and forces today's date if the backup is stale.
   - `checkRestoreBanner()` shows a "restore available" banner on load if a previous backup timestamp (`mqLastBackup` key) exists.
   - `confirmResetAll()` wipes all localStorage keys used by the app.

## Conventions

- Keep new JS consistent with the existing style: `var`, plain function declarations, inline event handlers via `addEventListener`, and the `el()`/direct DOM APIs — no template literals-as-HTML, no framework, no bundler.
- New colors/theme values go in the CSS custom properties block (`:root { --sky, --sun, --green, --pink, --purple, ... }`) rather than hardcoded hex values, matching the existing palette.
- UI copy is Japanese; keep new user-facing strings in Japanese and consistent in tone with existing messages (casual, encouraging, emoji-forward).
- Note there are duplicate `.data-tools`, `.data-btn`, and `.notice-overlay`/`.notice-box`/etc. CSS rule blocks in `index.html` (later ones win) — be aware of this when editing those styles so you change the block that's actually taking effect (the later/second one).
