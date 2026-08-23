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
    { "id": "…", "title": "…", "artist": "…", "appleMusicUrl": "…", "artworkUrl": "…", "type": "album", "tags": "…", "addedAt": "YYYY-MM-DD" }
  ]
}
```
`type` is `'album'` or `'song'` — shown as an `LP` / `45` badge on the cover. `tags` is a free-form comma list.

## Key implementation details

- **Apple Music**: metadata comes from the public, CORS-enabled iTunes Search/Lookup API (`itunes.apple.com/search`, `/lookup?id=`) — no auth needed. `parseAppleMusicUrl()` pulls the numeric id out of a `music.apple.com` URL (either the path's trailing id, or `?i=` for a track); `fetchAppleMusicMeta()` looks it up. `doAppleSearch()` hits `/search` directly for the in-app search tab. Artwork is upsized by rewriting the `NNxNNbb` segment of `artworkUrl100`.
- **Discogs**: optional cover-art alternative via `api.discogs.com/database/search` (`type=release&format=Vinyl`). Works unauthenticated (rate-limited); an optional personal token, stored in `localStorage['vshelf_discogs_token']` (global, not per-profile — it's the user's own account), raises the limit.
- **GitHub push/pull**: same pattern as Repertoire — GitHub Contents API (GET for sha, PUT to commit); config in `localStorage[GH_CFG_KEY]`, one token/repo/branch/path per profile.
- **Shelf visual**: the wood-ledge look under each row of covers is a single `repeating-linear-gradient` on the grid container's background, sized to the cover+gap row height — not per-row DOM. The vinyl disc "peeking" behind each cover is a `::` sibling div (`.vinyl-peek`), not a pseudo-element on the cover itself, so it can sit at a lower z-index without clipping.
