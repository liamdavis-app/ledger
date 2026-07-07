# Ledger — project notes

Single-file `index.html` gym/strength-tracking PWA. No build step, no framework, no git (yet). Warm-paper aesthetic: paper `#F4F1EA`, card `#FCFAF6`, ink `#211F1A`, clay `#B14E2C` (primary action), moss `#33694C` (data/progress).

Working file: `index.html` in this folder. This is the only file that matters — edit it directly.

## Deploy workflow (manual every time — I can't do this part)
1. I edit `index.html` here.
2. You drag-drop `index.html` into the Netlify dashboard to deploy.
3. On your phone: fully close and reopen the app (not just background it) to clear cache. This is required — "changes not showing" is almost always a stale cache, not a bad deploy.

Netlify dashboard URL (`app.netlify.com`) ≠ the live app URL (`something.netlify.app`). Only the live URL goes on the home screen.

## Facts that cost real debugging time to learn — don't relitigate these
- Data lives in the phone's browser **localStorage**, keyed to the exact URL. It is never in the file itself, and deploying a new file does **not** touch stored data.
- iOS can silently clear PWA storage. If you ever re-add the app to the home screen, the **old shortcut may still exist pointing at the old storage** — check for a duplicate icon before assuming data is lost. (See Known Incidents.)
- `load()` only falls back to `SEED` when localStorage is empty — it never wipes existing data. Safe to re-run.

## Architecture
- `db = {library, splits, sessions, active, prefs}` → persisted to localStorage key `ledger_v1`.
- `load()`: reads localStorage → merges hardcoded `BACKFILL` sessions array (dedup by `id`, safe to re-run) → runs a repair migration that ensures every session's exercises exist in the library, guessing group via `guessGroup()` for any that don't.
- `SEED` (~85 entries after latest update): `['Exercise Name','Group']` pairs, grep `var SEED=`.
- `BACKFILL` (13 sessions as of 2026-07-07): hardcoded past sessions with fixed ids, grep `var BACKFILL=`.
- `GROUPS`: Chest, Back, Shoulders, Biceps, Triceps, Legs, Core, Other.
- `guessGroup(name)`: keyword matcher in `GROUP_KEYWORDS`, ordered by specificity.
- Five tabs: Today, Splits, Library, History, Progress.
- Bottom sheets via `openSheet()`/`closeSheet()`, drag-to-dismiss (`attachSheetDrag()`), keyboard-aware resize via `attachKeyboardResize()` (VisualViewport API).

## Feature inventory (shipped)
- **Progress tab**: stats grid, headline insight card (`headlineInsight()`), collapsible accordion sections (state persisted to `db.prefs.accOpen`), timeframe selector (4wk/12wk/all-time), Lift explorer, Personal bests, Muscle group balance, Training frequency heatmap, weekly report card (Mon–Sun, dismissable, split + exercise-level deltas).
- **Lift explorer** (2026-07-08 redesign, `renderExplorer()`): picker collapses to a compact "[exercise] · Change ›" header once a lift is selected (`_pPickerOpen`), instead of always showing the full filterable list above the chart. Last 6 picks persist to `db.prefs.recentLifts`; a "Recent" chip row gives one-tap re-selection both in the collapsed header and the expanded picker. Split + muscle filters are tucked behind a single "Filters" toggle chip (`_pFiltersOpen`) that shows the active filter names even when collapsed, instead of three always-visible filter rows. Session log under the chart previews 5 entries with a "Show all N sessions" / "Show less" toggle (`_pLogExpanded`). 4 chart modes (1RM/top weight/top reps/kg-per-rep) and the SVG line chart with projection are unchanged.
- **Library**: description field per exercise, live smart group-suggestion on new exercise names, session rename.
- **Picker (Add exercise)**: keyboard never covers the sheet, single clear "Create '[name]'" action, "create and add to session in one step" flow.
- **Rest timer**: removed entirely (was covering other buttons) — no trace left in code.
- **App icon**: "Descending bars" concept, base64-embedded PNG in `<link rel="apple-touch-icon">`, no separate asset file.
- **Live in-session stats**: live PB comparison vs all-time history, live kg/rep with % delta vs previous session, both recompute automatically on `toggleDone()`.

## Known incidents
- **2026-07 data loss (resolved)**: re-adding the app to the home screen after a deploy created a second shortcut pointing at fresh/empty storage, while the original shortcut still held the real sessions. Recovered via History → Export from the old shortcut. Verified recovery: all 13 sessions cross-checked byte-for-byte (id/date/name) against `~/Downloads/ledger-2026-07-02.json` — confirmed complete as of 2026-07-07.
- Takeaway: if data ever looks missing, check for a duplicate home-screen icon before assuming it's gone.

## Open / in progress
- **Auto-backup feature** — not shipped. Plan: `autoBackup(silent)` downloads a dated JSON backup on `finishSession()` tap (iOS requires a user gesture, so it's tied to "Finish & log"); `prefs.lastBackup` timestamp; `lastBackupText()` helper for "Last backup: N days ago"; also wants a "Back up now" button added to the existing History tab Backup card (which already has Export/Import). None of this is in the current file — needs building from scratch, not just verifying.
- **GitHub / version history** — done as of 2026-07-08. Repo pushed to [github.com/liamdavis-app/ledger](https://github.com/liamdavis-app/ledger) (public). Deploy workflow unchanged: still drag-drop `index.html` into Netlify manually; git is purely for version history alongside that. Note: it would *not* have prevented the data-loss incident above (that was a browser-storage-vs-app-code issue, unrelated to how the code is versioned).

## Standard workflow for changes (keeps token use down — follow this, don't re-derive it)
1. Read only the relevant section of `index.html` before editing (it's long — don't read the whole file unless necessary).
2. Edit with targeted `Edit` calls.
3. Syntax check: extract the `<script>` block and run `new Function(js)` in Node.
4. Duplicate-function-name check via regex on `function NAME(`.
5. Logic smoke test (session counts, group assignments, id dedup) via a small Node script — cheaper than Playwright for data-model changes. Use Playwright only for actual UI/rendering changes.
6. Remind the user: deploy to Netlify, then fully close and reopen the app on the phone.
7. Update this file's "Feature inventory" / "Open / in progress" sections to reflect what shipped.

## Non-app context (only relevant if asked)
CV/LinkedIn framing of this project was discussed in an earlier chat session but not this repo — ask the user if they want that revisited, don't assume.
