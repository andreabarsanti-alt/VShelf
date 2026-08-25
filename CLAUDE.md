# CLAUDE.md

## Project Overview

vShelf is a single-page web app: two virtual shelves — a vinyl shelf for an Apple Music collection, and a movie shelf for streaming titles — switched via a header tab. Same architecture family as the `Repertoire` project — single HTML file, JSON data, localStorage + GitHub push/pull, multiple profiles.

## Architecture

- **Single-file app**: all HTML/CSS/JS lives in `index.html`. No build step, no frameworks.
- **Storage**: `localStorage`, seeded from a per-profile JSON file on first load. Each of the two shelves (Music, Movies) has its own storage key and seed file.
- **Profiles**: `?profile=<name>` in the URL (default `main`) namespaces the `localStorage` keys and the seed filenames — `vshelf_<name>.json` (Music), `vshelf_movies_<name>.json` (Movies). The profile *registry* (which profiles exist) lives unnamespaced in `localStorage['vshelf_profiles']`, so every profile's page load can see it; falls back to `DEFAULT_PROFILES` if empty.
- **Hosting**: static files via GitHub Pages.

## Files

| File | Description |
|------|--------------|
| `index.html` | Entire app |
| `vshelf_main.json` | Seed data for the default (`main`) profile's Music shelf |
| `vshelf_movies_main.json` | Seed data for the default (`main`) profile's Movies shelf |

## Data format

Music (`records`):
```json
{
  "records": [
    { "id": "…", "title": "…", "artist": "…", "appleMusicUrl": "…", "artworkUrl": "…", "type": "album", "owned": "no", "year": 2015, "tags": "…", "rating": 3.5, "addedAt": "YYYY-MM-DD" }
  ]
}
```
`type` is `'album'`, `'single'`, or `'compilation'` (default `'album'`) — not shown on the cover, just carried through from where the record was added and used for filtering/sorting. `owned` is `'no'` (default), `'vinyl'`, `'cd'`, or `'other'` — whether a physical copy is owned, likewise not shown on the cover, just used for filtering/sorting. `year` is the release year (number, or absent). `tags` is a free-form comma list. `rating` is `0`–`5` in `0.5` steps (`0`/absent = unrated).

Movies (`movies`):
```json
{
  "movies": [
    { "id": "…", "title": "…", "streamingUrl": "…", "imdbUrl": "…", "posterUrl": "…", "service": "netflix", "watched": "no", "year": 2015, "tags": "…", "rating": 3.5, "imdbRating": 8.1, "addedAt": "YYYY-MM-DD" }
  ]
}
```
`streamingUrl` is where it's actually watched (Netflix/Prime/etc.) — that's what opens on cover click, analogous to `appleMusicUrl`. `imdbUrl` is only ever used to fetch metadata (see below), never opened directly. `service` is one of `netflix`/`prime`/`disney`/`max`/`hulu`/`appletv`/`paramount`/`other`. `watched` is `'no'` (default) or `'yes'` — the movie-shelf analog of `owned`. `rating` is the same personal `0`–`5` half-step field as Music; `imdbRating` is a separate, read-only `0`–`10` number carried straight from OMDb, never user-edited.

## Key implementation details

- **Shelf switching**: a header tab (`.shelf-tab`, `switchShelf('music'|'movies')`) toggles which of `#music-panel`/`#movie-panel` is visible and sets `currentShelf`; the choice persists in `localStorage['vshelf_active_shelf']` (global, like edit mode). The two shelves have genuinely different record schemas, so filtering/sorting/rendering/the add-edit modal are separate, parallel implementations per shelf (`filteredSortedRecords`/`filteredSortedMovies`, `renderShelf`/`renderMovieShelf`, `openEditModal`/`openEditMovieModal`, etc. — Movies' names are the Music ones with "Movie"/`m` swapped in). What *is* shared is the schema-agnostic plumbing — GitHub push/pull, export/import, force-refresh — written once and driven by `shelfCtx()`, which returns `MUSIC_CTX` or `MOVIE_CTX` (each a `{storageKey, seedFile, ghCfgKey, arrayKey, getDb, setDb, render}` bundle) depending on `currentShelf`. Push/Pull/Refresh/Export/Import/GitHub-settings in the header and ⋯ menu are the same buttons regardless of which shelf is open — they just act on whatever `shelfCtx()` currently points at. The repo-sync-diff banner (`checkRepoSync`/`#sync-banner`) is Music-only for now; Movies doesn't check for repo drift on load.
- **Apple Music**: metadata comes from the public, CORS-enabled iTunes Lookup API (`itunes.apple.com/lookup?id=`) — no auth needed. `parseAppleMusicUrl()` pulls the numeric id out of a `music.apple.com` URL (either the path's trailing id, or `?i=` for a track); `fetchAppleMusicMeta()`/`fetchFromLink()` look it up and fill the form, including `year` from `releaseDate`. `type` is inferred from the lookup response since iTunes has no direct "single" field: a track-level link (`?i=`) is treated as a single, a collection with `collectionType === 'Compilation'` as a compilation, else album — always editable by hand afterward. Artwork is upsized by rewriting the `NNxNNbb` segment of `artworkUrl100`. There used to be an in-app "Search Apple Music" tab (`itunes.apple.com/search`) for adding without a link, but its relevance ranking was unreliable enough for well-known albums (some real titles never surfaced at all against a flood of unrelated same-titled singles) that it was dropped — paste-link is the only add path now. Discogs was tried as a secondary cover-art source and later dropped too.
- **IMDb / Movies**: IMDb itself has no public API, so metadata (title, year, poster, genre, `imdbRating`) comes from OMDb (`omdbapi.com`), a free key-gated JSON API that wraps IMDb's data — the key lives in `localStorage['vshelf_omdb_key']` (global, entered via ⋯ → "OMDb API key…"). Unlike Apple Music, a pasted Netflix/streaming link carries no IMDb id to resolve, so the Movies add/edit modal has two separate link fields: **IMDb Link** (`parseImdbUrl()` pulls the `tt…` id out of an `imdb.com/title/...` URL; `fetchImdbMeta()`/`fetchFromImdbLink()` look it up via OMDb and fill title/year/poster/tags/`imdbRating` — used only for the lookup, never stored as the "open" link) and **Streaming Link** (`streamingUrl`, pasted directly, required, no fetch behavior — it's just where the movie actually opens from on cover click).
- **GitHub push/pull**: same pattern as Repertoire — GitHub Contents API (GET for sha, PUT to commit) via the shared `githubGetFile()`/`githubPutFile()` helpers; config in `localStorage[ctx.ghCfgKey]`, one token/repo/branch/path per profile *and* per shelf (Music and Movies push to independent files/targets by default). **🔄 Refresh** (`forceRefresh`) is the header's always-visible, no-token hard-refresh — refetches the active shelf's seed file and hard-reloads `index.html` itself (cache-busted), same `hardReloadPage()` mechanism as Repertoire.
- **Edit mode**: a header checkbox (`localStorage['vshelf_edit_mode']`, global — not per-profile) toggles a `body.edit-mode` class. There are no per-cover icons — `handleCoverClick()` checks `body.edit-mode` at click time: with it on, clicking a cover opens the edit modal (`openEditModal`, whose footer has the only "🗑 Remove" button); with it off, clicking opens the record's Apple Music link (`openRecord`) instead. **＋ Add Record** is unaffected and always available. This is the only path into the edit modal, so with edit mode off records can't be edited or removed at all.
- **Shelf visual**: the wood-ledge look under each row of covers is a single `repeating-linear-gradient` on the grid container's background, sized to the cover+gap row height — not per-row DOM. The vinyl disc "peeking" behind each cover is a `::` sibling div (`.vinyl-peek`), not a pseudo-element on the cover itself, so it can sit at a lower z-index without clipping.
- **Rating**: no star glyphs anywhere in the UI — dropped after repeated, unfixable-in-that-form iOS/iPadOS home-screen Safari (WKWebView) rendering bugs (a `★★★★★` fill clipped via a CSS-custom-property width, then the same fill with an inline width, both still rendered every rating as a full 5 stars there). `rating` is still a `0`–`5` half-step number under the hood — settable via the edit form's `#rf-rating` range input (carries half-star units `0`–`10`; `rating = value / 2`, shown as plain text like "3.5 / 5" via `#rf-rating-label`) and usable for filtering/sorting; grouped shelves (below) are how it's surfaced on the shelf itself now instead of a per-cover badge.
- **Filter/sort**: toolbar dropdowns filter by format, owned, artist, year, tag, and rating (all populated from distinct values in the shelf, except format and owned which are fixed enums). The owned dropdown's blank option is "Any" (no filter); `owned` filters to any physical copy (`owned !== 'no'`), separate from the exact `vinyl`/`cd`/`other`/`no` options. The search box covers title/artist/tag text. Sort adds format, owned, year, and tags to the usual title/artist/rating/date-added options. Every sort mode groups the shelf into separate `.shelf-group` sections by whatever field it primarily sorts on — a labeled header per distinct value (e.g. one shelf per rating, per year, per format, per artist) — except the two "added" sorts, which stay one flat shelf since chronological order has no natural bucket. `groupRecords()` in `renderShelf()`'s call chain just chunks the already-sorted list by consecutive equal key (via `groupKey()`/`groupLabel()`, keyed off `sortMode`), it doesn't re-sort or re-bucket independently — the grouping is only correct because it relies on the primary sort key being contiguous, which `filteredSortedRecords()`'s comparators (primary key first, secondary tie-breaker after) guarantee. Movies has its own parallel `filteredSortedMovies()`/`groupMovies()`/`groupKey`↔`movieGroupKey` etc. following the identical rule (group by whatever `movieSortMode` is sorting on, except the two "added" sorts) against its own fields (service, watched, year, tags, rating).
