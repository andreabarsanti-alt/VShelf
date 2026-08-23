# vShelf

A virtual vinyl shelf for your Apple Music collection — a single-page web app, no backend or build step required.

You don't own physical records; you "acquire" a record by pasting its Apple Music link and it takes its place on your shelf. Click a cover to open it in Apple Music.

## Features

- **Two shelf views** — the plain wood-shelf grid (each cover with a little vinyl disc peeking out behind it), or a **crate view**: flip through records as thin spines packed side by side, hover/tap one to pop its full cover up. Click a cover to open it in Apple Music.
- **Add a record** — paste a `music.apple.com` album or song link and it auto-fills title, artist, cover art and release year via Apple's public catalog lookup
- **Format** — Album, Single, or Compilation (guessed from the Apple Music link, editable)
- **Rating** — rate any record 0–5 stars in half-star steps; sort the shelf by rating
- **Edit mode** — a header checkbox that reveals each record's edit/remove icons; leave it off to browse without the risk of accidental edits (adding records is always available)
- Tag records freely (genre, mood, whatever) and filter/search/sort the shelf by format, artist, title, year, tags, or rating
- **Data** stored in `localStorage`, seeded from `vshelf_<profile>.json` on first load
- **⬆ Push / ⬇ Pull** — sync the shelf directly with this GitHub repo via the GitHub Contents API (needs a personal access token, configured in-app)
- **🔄 Refresh** — hard-reload the app and its data from what's published, bypassing the browser cache
- **Export / Import** — download or load a `.json` backup
- **Profiles** — keep multiple separate shelves (e.g. different people, different genres) via the header dropdown; each is its own `vshelf_<id>.json`

## Hosting on GitHub Pages

1. Push `index.html` and `vshelf_main.json` to the `main` branch
2. Repo Settings → **Pages** → source: `main` / `root`
3. Visit `https://<username>.github.io/<repo>/`

Works fully offline after the first load.

## Data format

```json
{
  "records": [
    {
      "id": "unique-id",
      "title": "Album Title",
      "artist": "Artist Name",
      "appleMusicUrl": "https://music.apple.com/us/album/...",
      "artworkUrl": "https://...",
      "type": "album",
      "year": 1971,
      "tags": "jazz, 1970s",
      "rating": 3.5,
      "addedAt": "2026-08-23"
    }
  ]
}
```
