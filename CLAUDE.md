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
    { "id": "…", "title": "…", "artist": "…", "appleMusicUrl": "…", "artworkUrl": "…", "type": "album", "year": 2015, "tags": "…", "rating": 3.5, "addedAt": "YYYY-MM-DD" }
  ]
}
```
`type` is `'album'`, `'single'`, or `'compilation'` (default `'album'`) — not shown on the cover, just carried through from where the record was added and used for filtering/sorting. `year` is the release year (number, or absent). `tags` is a free-form comma list. `rating` is `0`–`5` in `0.5` steps (`0`/absent = unrated).

## Key implementation details

- **Apple Music**: metadata comes from the public, CORS-enabled iTunes Lookup API (`itunes.apple.com/lookup?id=`) — no auth needed. `parseAppleMusicUrl()` pulls the numeric id out of a `music.apple.com` URL (either the path's trailing id, or `?i=` for a track); `fetchAppleMusicMeta()`/`fetchFromLink()` look it up and fill the form, including `year` from `releaseDate`. `type` is inferred from the lookup response since iTunes has no direct "single" field: a track-level link (`?i=`) is treated as a single, a collection with `collectionType === 'Compilation'` as a compilation, else album — always editable by hand afterward. Artwork is upsized by rewriting the `NNxNNbb` segment of `artworkUrl100`. There used to be an in-app "Search Apple Music" tab (`itunes.apple.com/search`) for adding without a link, but its relevance ranking was unreliable enough for well-known albums (some real titles never surfaced at all against a flood of unrelated same-titled singles) that it was dropped — paste-link is the only add path now. Discogs was tried as a secondary cover-art source and later dropped too.
- **GitHub push/pull**: same pattern as Repertoire — GitHub Contents API (GET for sha, PUT to commit); config in `localStorage[GH_CFG_KEY]`, one token/repo/branch/path per profile. **🔄 Refresh** (`forceRefresh`) is the header's always-visible, no-token hard-refresh — refetches `SEED_FILE` and hard-reloads `index.html` itself (cache-busted), same `hardReloadPage()` mechanism as Repertoire.
- **Edit mode**: a header checkbox (`localStorage['vshelf_edit_mode']`, global — not per-profile) toggles a `body.edit-mode` class. The per-record edit/remove icons only exist in the DOM's hoverable state when that class is present (`.cover-actions` is `display:none` otherwise); **＋ Add Record** is unaffected and always available. This is the only path into the edit modal, so with edit mode off records can't be edited or removed at all.
- **Shelf visual**: the wood-ledge look under each row of covers is a single `repeating-linear-gradient` on the grid container's background, sized to the cover+gap row height — not per-row DOM. The vinyl disc "peeking" behind each cover is a `::` sibling div (`.vinyl-peek`), not a pseudo-element on the cover itself, so it can sit at a lower z-index without clipping.
- **Crate view**: an alternative to the plain grid (toggle in the header, `localStorage['vshelf_view_mode']`, global). Records render as thin vertical spines (`.spine`) in a horizontally-scrolling row, tinted by a deterministic hash of the id (`hashHue()` — a stand-in for real cover color extraction) rather than actual artwork color. Hovering/focusing a spine pops the full cover art above it via `.spine-pop`, which reuses the exact same `.cover` markup (factored out as `coverInnerHTML()`) as the grid card, so edit/remove/open behavior is identical in both views. The crate container gets a fixed `min-height` rather than `overflow-y:visible` to hold the popped card — `overflow-x:auto` would otherwise force `overflow-y` to compute as `auto` too (per spec, differing axes can't mix in `visible`), which would clip the pop-up.
- **Rating**: `.star-rating` renders a dim `★★★★★` background with a colored `★★★★★` copy absolutely positioned on top and clipped via `width:var(--pct)` — a true partial-character fill (so half-star widths look right), not a special half-star glyph. The edit form's `#rf-rating` range input carries half-star units (`0`–`10`); `rating = value / 2`.
- **Filter/sort**: toolbar dropdowns filter by format, artist, year, tag, and rating (all populated from distinct values in the shelf, except format which is the fixed enum); the search box covers title/artist/tag text. Sort adds format, year, and tags to the usual title/artist/rating/date-added options.
