# vShelf

A virtual vinyl shelf for your Apple Music collection — a single-page web app, no backend or build step required.

You don't own physical records; you "acquire" a record by pasting its Apple Music link (or finding it via search) and it takes its place on your shelf. Click a cover to open it in Apple Music.

## Features

- **Shelf view** — records displayed as album covers on a wood-shelf grid, each with a little vinyl disc peeking out behind it. Click a cover to open it in Apple Music.
- **Add a record**
  - **Paste a link** — drop in a `music.apple.com` album or song URL and it auto-fills title, artist and cover art via Apple's public catalog lookup
  - **Search Apple Music** — search the catalog by name and pick a result directly, no link needed
  - **Discogs cover lookup** — optionally pull an alternate (often higher-quality vinyl scan) cover from Discogs instead of Apple's own artwork
- Edit or remove any record; tag records freely (genre, mood, whatever) and filter/search/sort the shelf by them
- **Data** stored in `localStorage`, seeded from `vshelf_<profile>.json` on first load
- **⬆ Push / ⬇ Pull** — sync the shelf directly with this GitHub repo via the GitHub Contents API (needs a personal access token, configured in-app)
- **🔄 Refresh** — reload the app and its data from what's published, bypassing the browser cache
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
      "tags": "jazz, 1970s",
      "addedAt": "2026-08-23"
    }
  ]
}
```
