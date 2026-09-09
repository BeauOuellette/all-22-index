# all-22-index

Builds a compact, queryable SQLite index of NFL play-by-play and publishes it to
GitHub Releases. A GitHub Action rebuilds it daily and cuts a new release only
when the data actually changed.

Consumers download one gzipped file and query it locally — no server, no API key.

## What's in it

| | |
|---|---|
| source | [nflverse](https://github.com/nflverse/nflverse-data) play-by-play + FTN charting |
| rows | ~48,800 plays per season |
| columns | 337 |
| size | ~64 MB, **~15 MB gzipped** |

Columns come from two places:

* **nflverse play-by-play** — participants, situation, EPA, win probability, and
  the ~43 `*_player_id` columns that say who did what on each snap.
* **FTN charting** (2022+, published free by nflverse) — play action, RPO,
  screen, motion, pressure counts, box count, drops, throwaways. Joined on
  `nflverse_game_id` + `nflverse_play_id`.

A `players` table maps `gsis_id` → `nfl_id` with position, team and a count of
how many indexed plays each player appears in.

57 low-value columns are dropped by default (running score/EPA totals,
per-outcome probability splits, venue metadata) — that's what takes a season
from 101 MB to 64 MB. Pass `--full` to keep them.

## Build it yourself

Python 3.9+, standard library only. No dependencies.

```bash
python3 build_index.py 2025          # one season
python3 build_index.py 2022 2023 2024 2025
python3 build_index.py 2025 --full   # keep all 372 columns
```

Output lands in `data/dist/`:

* `plays_<seasons>.db.gz` — the index
* `manifest.json` — `sha256`, byte size, row/column counts, seasons

## Consuming a release

```
https://github.com/OWNER/all-22-index/releases/latest/download/manifest.json
```

Fetch the manifest, compare `sha256` against your cached copy, and download
`file` from the same directory only when it differs. Verify the checksum before
opening it.

```jsonc
{
  "file": "plays_2025.db.gz",
  "sha256": "…",        // of the .db.gz — verify your download against this
  "content_hash": "…",  // of the source data — use this to detect real changes
  "bytes": 15004718,
  "rows": 48771,
  "seasons": [2025],
  "columns": 337
}
```

Two hashes, deliberately. `sha256` covers the compressed artifact, so a client can
verify what it downloaded. `content_hash` covers the nflverse source files, so it
only moves when the underlying data does — the gzip and SQLite bytes shift with
the zlib and SQLite versions on the build machine, which would otherwise look
like a change on every runner-image update.

## Schedule

Rebuilds daily at 09:17 UTC, after nflverse's own jobs. nflverse republishes the
whole season asset in place rather than shipping per-game deltas, so the job
rebuilds, compares the hash to the published manifest, and **only cuts a release
when the result differs**. Most offseason runs are a no-op.

Past seasons are immutable once final — cache them forever. Only the current
season changes.

## Licence

The code here is MIT. The data is nflverse's and FTN's; see their terms.
