# Lesson 02 — Data Flow

## What this system does

SoundTræk takes two independent streams of historical data — running activity files and Spotify playback exports — and joins them by timestamp to answer: *what was playing at each metre of each run?*

This lesson traces how raw data files become a colored route map.

---

## Evidence

- `core/models.py` — all seven models
- `docs/schema.md` — full schema reference
- `meta/project_manifest.json`
- Source data: `~/Library/CloudStorage/Dropbox/Projects/`
  - 447 `*_spotify_data.csv` files
  - ~300 `activity-{uuid}.json` files (Nike Run Club)
  - 233 `*.fit` files (Garmin)

---

## Walkthrough

### The full pipeline

```
Raw Files (Dropbox)
        │
        ├── Spotify CSVs ──────────────► Step 1: import_spotify
        │   447 files, 2023–2026             │
        │                                    ▼
        │                          Album, Artist, Track, PlayEvent
        │
        └── Running Files ─────────────► Step 2: import_runs
            NRC JSON (~300)                  │
            Garmin .fit (233)                ▼
            Strava CSV                  Run, RunPoint
                                             │
                                             │
                        ┌────────────────────┘
                        │
                        ▼
              Step 3: match_runs
              (time-based join)
                        │
                        ▼
                RunTrackMatch
           (which song, at which metre)
                        │
                        │
                        ▼
              Step 4: extract_album_colors
              (fetch album art → colorthief)
                        │
                        ▼
              Album.palette_json
              Track.assigned_color_hex
                        │
                        │
                        ▼
              Frontend: colored route map
```

### Step 1 — Spotify import

Each CSV file (`20230101-161506_spotify_data.csv`) is one session's listening export. Each row is one track play.

Key parsing detail: the `track` column is a Python dict serialized as a string with single quotes. It must be parsed with `ast.literal_eval()`, not `json.loads()`.

From each row we extract and upsert:
- `Album` — by `spotify_id` (inside `track['album']['id']`)
- `Artist` — by `spotify_id` (for each artist in `track['artists']`)
- `Track` — by `spotify_id` (`track['id']`), linked to Album and Artists
- `PlayEvent` — one row per play, deduplicated on `(user, played_at, track_id)`

### Step 2 — Running import

**Nike Run Club (NRC JSON):**
Each `activity-{uuid}.json` has `start_epoch_ms` (milliseconds), a `metrics` array (one entry per metric type), and `summaries`. The `values` array inside each metric is the time-series.

```
{
  "id": "8f87266e-...",
  "start_epoch_ms": 1673714720746,
  "metrics": [
    { "type": "distance", "values": [{"value": 0.0, "start_epoch_ms": ...}, ...] },
    { "type": "latitude",  "values": [...] },
    ...
  ]
}
```

**Garmin .fit:**
Binary format, parsed with the `fitparse` library. Yields `session` messages (summary) and `record` messages (per-second samples). GPS coordinates are stored in **semicircles** — convert to degrees: `degrees = semicircles * (180 / 2**31)`.

Both sources produce `Run` (summary) and `RunPoint` (time-series) records.

### Step 3 — Time-based join

This is the core of SoundTræk. For each run, we find which Spotify play events fall within its time window, then use `pandas.merge_asof` to assign each run point the most recent track that started playing before that moment.

```python
# Conceptually:
merged = pd.merge_asof(
    run_points.sort_values('timestamp'),
    play_events.sort_values('played_at'),
    left_on='timestamp',
    right_on='played_at',
    direction='backward'
)
```

Result: every run point knows which track was playing. This gets stored as `RunTrackMatch` rows — one per track per run, with `distance_at_start_m` and `percent_run` (0.0–1.0).

### Step 4 — Color extraction

For each Album where `palette_fetched_at IS NULL`:
1. Download the image from `art_url`
2. Run `colorthief` to extract a palette of up to 8 colors
3. Filter out near-neutral colors (very light, very dark, very grey)
4. Store the palette in `Album.palette_json`
5. Set `Album.dominant_color_hex` to the most vibrant qualifying color
6. For each Track in the album, set `assigned_color_hex = palette[track_number % len(palette)]`

This ensures tracks from the same album get visually related but distinct colors — important when you listen to multiple songs from the same album during a run.

### Frontend output (not yet built)

The colored route is produced by:
1. Querying `RunPoint` records for a run (ordered by timestamp)
2. For each point, looking up its `RunTrackMatch` to get `play_event → track → assigned_color_hex`
3. Rendering the GPS coordinates as a polyline, segmented and colored by `assigned_color_hex`

---

## Visual artifact

```
Run timeline:
|--[Song A]--|--[Song B]--|--[Song B]--|--[Song C]--|
0%          25%          50%          75%         100%

Route (plan view):
●━━━━━━━━━━━━◉━━━━━━━━━━━━◉━━━━━━━━━━━━◉━━━━━━━━━━━●
  #4A6FD4 (A)  #7ECFC0 (B)  #7ECFC0 (B)  #E8A24C (C)
                  ↑ same album, different track = different shade
```

---

## Design rationale

**Why a separate color extraction step?** Fetching and processing ~500 album art images takes several minutes. Running it as a separate idempotent command means the import pipeline stays fast, and color extraction can be run once and cached forever.

**Why `merge_asof` and not an exact join?** Spotify `played_at` is when a track finished, not when it started. Running GPS samples fire every second. There's no guaranteed exact timestamp match — a nearest-prior join is the correct operation.

**Why store `percent_run` on `RunTrackMatch`?** It decouples the map renderer from having to re-compute progress through the run. The match table is a pre-computed join result.

---

## Learning takeaway

Data enters SoundTræk from two independent sources, gets normalized into a shared schema, and is joined by time. The entire value of the app lives in Step 3 — the merge. Everything before it is ingestion and normalization; everything after it is presentation.

---

*Did this section make sense?*
*[ Got it ] [ Needs more detail ] [ Still confused ]*
*Record feedback in `meta/comprehension_signals.json`*
