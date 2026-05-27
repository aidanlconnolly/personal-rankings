# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file personal-rankings website (`index.html`) — Aidan's ranked lists of movies, books, TV shows, and movie series. Vanilla HTML/CSS/JS with no build step, no dependencies, no package.json. Open `index.html` directly in a browser, or serve the folder with any static server.

Deployed to Vercel at `personal-rankings.vercel.app`; `git push origin main` triggers auto-deploy.

## Run / preview locally

```bash
# Anywhere a static server works
python3 -m http.server 5700
# then http://localhost:5700/
```

`open index.html` also works for everything except APIs that require a real `http(s):` origin (Wikipedia/TVMaze/Google Books all work over `file://` in practice).

## Architecture

Everything lives in `index.html`. The script has these layered concerns, in order:

### 1. Data (`DATA`, `TABS`, `SECTION_TO_TAB`)
- `DATA` is keyed by **section key** (`bookSeries`, `tvSeries`, `movies`, `movieSeries`, `booksToRead`, `tvToWatch`, `moviesToWatch`). Each section has `{label, type, items, watchlist?}`.
- Items are built with the `m(title, ...tags)` helper → `{title, tags}`. Title is the display string (exact spelling preserved from source doc, including odd casing like "Shashank Redemption"). Tags drive the filter chips.
- `TABS` defines the **4 visible tabs** (Movies, Books, TV Shows, Movie Series), each pointing at one or two sections (`[primary, watchlist?]`). Tab order in this array = display order in the nav. The "★ My Saved" tab is **not** in `TABS` — it's special-cased in `activateTab` and renders its own dynamic view.
- `SECTION_TO_TAB` is the reverse lookup, built once at module load.

### 2. Cover fetching (`fetchTV`, `fetchMovie`, `fetchBookCover`, `lookupCover`)
Three different free, no-API-key sources chosen for CORS-friendliness:
- **TV shows** → TVMaze (`api.tvmaze.com/singlesearch/shows`)
- **Movies** → Wikipedia REST API: tries `${title}_(film)` direct, then `${title}`, then falls back to a search query + top-4 walk. Skips disambiguation pages and non-image results.
- **Books** → Google Books volumes API with Open Library as fallback.

Results are cached forever in `localStorage` under `cover:v2:${type}:${title}`. The version prefix (`v2`) is the cache-bust knob — bump it to force re-fetch of everything. For targeted invalidation, the `init()` function has a small list of keys it removes on every page load (used to fix specific covers after override changes).

### 3. Title-disambiguation overrides (`SEARCH_OVERRIDES`)
Plain object whose keys are either the display title (`'Wolf of Wall street'`) or `'type:title'` for items that exist in multiple sections (`'movie:Percy Jackson'` vs `'tv:Percy Jackson'`). The type-prefixed key wins when both exist. When adding a new item that searches to the wrong Wikipedia page, add an override here — usually a more specific query like `'The Wolf of Wall Street (2013 film)'` or `'Casino Royale (2006 film)'`.

### 4. Saved list (`SAVED`, `toggleSaved`, `renderSavedView`)
- Stored in `localStorage` under `saved:v1` as `{movies: [], books: [], tv: [], movieSeries: []}` (keyed by tab key).
- Each saved ID is `${tabKey}::${sectionKey}::${title}` — composite so the same title saved from different sections stays distinct.
- The "My Saved" view is rebuilt from scratch every time it's opened (via `renderSavedView`) and re-uses `buildCardEl` to render the cards. Covers are re-hydrated from cache on demand.

### 5. Rendering & layout
- `buildCardEl` returns a card with: dark title strip on top (badge top-left, save button top-right, title centered), then poster cover below. Top-3 ranks (gold/silver/bronze) wrap the inner content in an animated gradient border via the `.tier-1/2/3` classes; they have an extra `.inner` wrapper for the rounded interior — preserve this structure when editing.
- `applyZigzag(grid)` is the boustrophedon layout: cards staircase down within each row, and rows alternate direction. It runs on init and on window resize (debounced). It explicitly sets `gridColumn`, `gridRow`, and `marginTop` per card. Hidden-by-filter cards are skipped so the path stays continuous. The grid uses `align-items: start` so shorter cards in a row don't stretch.

### 6. Filter chips
Built from the **active section's** tags (not the whole tab) every time `activateTab` runs. Multi-select; OR semantics (a card shows if *any* of its tags match *any* selected chip). `applyFilter` toggles a `hidden-by-filter` class and then re-runs `applyZigzag` so the path collapses around hidden cards.

### 7. Tab activation (`activateTab`)
Three branches:
- `key === 'saved'` → hide all `.list` sections, show the special `#saved-view` section, hide filters + subtabs.
- Otherwise → activate the section corresponding to the tab's current sub-view index (tracked in `subviewPerTab`), rebuild subtabs + filter chips.

The sub-tab segmented control (Top Picks ⇄ Want to Watch) shows only for tabs whose `sections` array has length > 1.

### 8. Theatre curtain backdrop
Pure CSS in `.curtain-bg` — a fixed-position layer behind the hero containing a pelmet, two `.curtain` halves (each with vertical pleat folds via `repeating-linear-gradient`), a center seam shadow, and CSS `@keyframes` `curtain-sway-left/right` for the subtle skew. A scroll listener in `init()` linearly fades the whole `.curtain-bg` from opacity 1 → 0 over the first 480px of scroll.

## Editing conventions worth knowing

- **Adding an item to a list**: append to the relevant `DATA[section].items` array as `m('Title', 'Tag1', 'Tag2')`. Preserve the user's exact spelling from the source doc (`Best book series of all time.docx` in `~/Desktop/Personal/Personal rankings/`).
- **Changing rank order**: just reorder the items in the array — the rank badge is `idx + 1`.
- **A poster looks wrong**: add or update an entry in `SEARCH_OVERRIDES` (use `'type:title'` if the title appears in multiple sections), then add the cache key to the `init()` force-refetch list so existing visitors re-fetch on next load.
- **A new tag should show as a filter chip**: just use it in an item. Chips are derived from the section's tags automatically.
- **CSS font/color overhaul**: heading/title font-family is `'Playfair Display'`; body is `'EB Garamond'`. Both pulled from Google Fonts in `<link>` at the top of `<head>`. Colors are CSS variables in `:root` near the top of `<style>`.
