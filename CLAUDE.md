# CLAUDE.md

## Project Overview

vShelf is a single-page web app: a virtual vinyl shelf for an Apple Music collection. Same architecture family as the `Repertoire` project — single HTML file, JSON data, localStorage + GitHub push/pull, multiple profiles.

## Architecture

- **Single-file app**: all HTML/CSS/JS lives in `index.html`. No build step, no frameworks.
- **Storage**: `localStorage`, seeded from a per-profile JSON file on first load.
- **Profiles**: `?profile=<name>` in the URL (default `main`) namespaces the `localStorage` key and the seed filename — `vshelf_<name>.json`. The profile *registry* (which profiles exist) lives unnamespaced in `localStorage['vshelf_profiles']`, so every profile's page load can see it; falls back to `DEFAULT_PROFILES` if empty.
- **Hosting**: static files via GitHub Pages.

## Files

| File | Description |
|------|--------------|
| `index.html` | Entire app |
| `vshelf_main.json` | Seed data for the default (`main`) profile |

## Data format

```json
{
  "records": [
    { "id": "…", "title": "…", "artist": "…", "appleMusicUrl": "…", "artworkUrl": "…", "type": "album", "owned": "no", "year": 2015, "tags": "…", "rating": 3.5, "addedAt": "YYYY-MM-DD" }
  ]
}
```
`type` is `'album'`, `'single'`, or `'compilation'` (default `'album'`) — not shown on the cover, just carried through from where the record was added and used for filtering/sorting. `owned` is `'no'` (default), `'vinyl'`, `'cd'`, or `'other'` — whether a physical copy is owned, likewise not shown on the cover, just used for filtering/sorting. `year` is the release year (number, or absent). `tags` is a free-form comma list. `rating` is `0`–`5` in `0.5` steps (`0`/absent = unrated).

## Key implementation details

- **Apple Music**: metadata comes from the public, CORS-enabled iTunes Lookup API (`itunes.apple.com/lookup?id=`) — no auth needed. `parseAppleMusicUrl()` pulls the numeric id out of a `music.apple.com` URL (either the path's trailing id, or `?i=` for a track); `fetchAppleMusicMeta()`/`fetchFromLink()` look it up and fill the form, including `year` from `releaseDate`. `type` is inferred from the lookup response since iTunes has no direct "single" field: a track-level link (`?i=`) is treated as a single, a collection with `collectionType === 'Compilation'` as a compilation, else album — always editable by hand afterward. Artwork is upsized by rewriting the `NNxNNbb` segment of `artworkUrl100`. There used to be an in-app "Search Apple Music" tab (`itunes.apple.com/search`) for adding without a link, but its relevance ranking was unreliable enough for well-known albums (some real titles never surfaced at all against a flood of unrelated same-titled singles) that it was dropped — paste-link is the only add path now. Discogs was tried as a secondary cover-art source and later dropped too.
- **GitHub push/pull**: same pattern as Repertoire — GitHub Contents API (GET for sha, PUT to commit); config in `localStorage[GH_CFG_KEY]`, one token/repo/branch/path per profile. **🔄 Refresh** (`forceRefresh`) is the header's always-visible, no-token hard-refresh — refetches `SEED_FILE` and hard-reloads `index.html` itself (cache-busted), same `hardReloadPage()` mechanism as Repertoire.
- **Edit mode**: a header checkbox (`localStorage['vshelf_edit_mode']`, global — not per-profile) toggles a `body.edit-mode` class. The per-record edit/remove icons only exist in the DOM's hoverable state when that class is present (`.cover-actions` is `display:none` otherwise); **＋ Add Record** is unaffected and always available. This is the only path into the edit modal, so with edit mode off records can't be edited or removed at all.
- **Shelf visual**: the wood-ledge look under each row of covers is a single `repeating-linear-gradient` on the grid container's background, sized to the cover+gap row height — not per-row DOM. The vinyl disc "peeking" behind each cover is a `::` sibling div (`.vinyl-peek`), not a pseudo-element on the cover itself, so it can sit at a lower z-index without clipping.
- **Rating**: no star glyphs anywhere in the UI — dropped after repeated, unfixable-in-that-form iOS/iPadOS home-screen Safari (WKWebView) rendering bugs (a `★★★★★` fill clipped via a CSS-custom-property width, then the same fill with an inline width, both still rendered every rating as a full 5 stars there). `rating` is still a `0`–`5` half-step number under the hood — settable via the edit form's `#rf-rating` range input (carries half-star units `0`–`10`; `rating = value / 2`, shown as plain text like "3.5 / 5" via `#rf-rating-label`) and usable for filtering/sorting; grouped shelves (below) are how it's surfaced on the shelf itself now instead of a per-cover badge.
- **Filter/sort**: toolbar dropdowns filter by format, owned, artist, year, tag, and rating (all populated from distinct values in the shelf, except format and owned which are fixed enums). The owned dropdown's blank option is "Any" (no filter); `owned` filters to any physical copy (`owned !== 'no'`), separate from the exact `vinyl`/`cd`/`other`/`no` options. The search box covers title/artist/tag text. Sort adds format, owned, year, and tags to the usual title/artist/rating/date-added options. Every sort mode groups the shelf into separate `.shelf-group` sections by whatever field it primarily sorts on — a labeled header per distinct value (e.g. one shelf per rating, per year, per format, per artist) — except the two "added" sorts, which stay one flat shelf since chronological order has no natural bucket. `groupRecords()` in `renderShelf()`'s call chain just chunks the already-sorted list by consecutive equal key (via `groupKey()`/`groupLabel()`, keyed off `sortMode`), it doesn't re-sort or re-bucket independently — the grouping is only correct because it relies on the primary sort key being contiguous, which `filteredSortedRecords()`'s comparators (primary key first, secondary tie-breaker after) guarantee.
