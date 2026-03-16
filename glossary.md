# Glossary

**Album** — A Spotify album record. Stores the extracted color palette. Shared across all users.

**assigned_color_hex** — The hex color assigned to a specific Track for route visualization. Computed as `palette[track_number % len(palette)]` after album color extraction.

**colorthief** — Python library that extracts dominant colors and a color palette from an image. Used to pull colors from Spotify album art URLs.

**dominant_color_hex** — The most vibrant, non-neutral color from an album's palette. Used as the headline color for that album.

**external_id** — The identifier from the original data source (e.g. NRC activity UUID, .fit filename). Combined with `source`, it uniquely identifies a run for idempotent import.

**FIT file** — Binary file format used by Garmin devices to record workout data. Contains `session` (summary) and `record` (per-second) messages. GPS coordinates are stored in semicircles.

**fitparse** — Python library for reading Garmin `.fit` files.

**idempotent** — An operation that can be run multiple times without changing the result after the first run. All SoundTræk import commands are idempotent.

**merge_asof** — A pandas function that performs a nearest-prior join on sorted timestamps. Used to match Spotify play events to run GPS points.

**NRC** — Nike Run Club. The older source of running data (~300 JSON activity files).

**palette_json** — JSON array of up to 8 hex color strings extracted from album art. Stored on `Album`. Used to assign colors to individual tracks.

**palette_fetched_at** — Datetime when album art colors were extracted. `NULL` means colors have not been extracted yet.

**percent_run** — A float between 0.0 and 1.0 representing how far into a run a track started playing. Used to position the track on the route map.

**PlayEvent** — One occurrence of a Spotify track being played. User-owned. Deduplicated on `(user, played_at, track)`.

**Run** — One running activity. User-owned. Summarizes a session from any source (NRC, Garmin, Strava).

**RunPoint** — One GPS/metrics sample within a run, typically one per second. Contains lat/lon, distance, speed, heart rate, cadence.

**RunTrackMatch** — The join between a Run and a PlayEvent. Stores `distance_at_start_m` and `percent_run`. Pre-computed at import time so the frontend doesn't need to re-join.

**semicircles** — The unit Garmin uses for GPS coordinates in `.fit` files. Convert to degrees: `degrees = semicircles * (180 / 2**31)`.

**Spotify ID** — Spotify's unique identifier for tracks, albums, and artists (a 22-character alphanumeric string). Used as primary key for `Track`, `Album`, and `Artist` tables.

**Supabase** — Managed Postgres with a REST API and Row Level Security. The planned production database for SoundTræk.

**Track** — A Spotify track (song). Global shared data. Has an `assigned_color_hex` for route visualization.
