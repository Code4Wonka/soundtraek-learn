# Lesson 03 — Storage Model

## What this system does

SoundTræk stores data in seven database tables across two categories: global shared reference data (music catalog) and per-user owned data (runs and listening history). This lesson explains each table, the decisions behind the schema design, and how the tables relate to each other.

---

## Evidence

- `core/models.py`
- `core/migrations/0001_initial.py`
- `soundtraek/settings.py` — database configuration
- `docs/schema.md` — full column-level reference

---

## Walkthrough

### Two categories of data

```
SHARED (no user FK)          USER-OWNED (user FK required)
─────────────────────        ─────────────────────────────
Album                        Run
Artist                       RunPoint
Track                        PlayEvent
                             RunTrackMatch
```

**Shared tables** store the music catalog. If 100 users all listened to the same Drake album, there's one `Album` row, one set of `Track` rows, one color palette — not 100 copies. Album art color extraction happens once for the whole platform.

**User-owned tables** store personal activity. Each user's runs and listening events are isolated by `user_id`.

---

### The music catalog: Album → Track → Artist

```python
# core/models.py

class Album(models.Model):
    spotify_id = models.CharField(max_length=64, primary_key=True)
    name = models.CharField(max_length=512)
    art_url = models.URLField(max_length=1024, blank=True)
    dominant_color_hex = models.CharField(max_length=7, blank=True)
    palette_json = models.JSONField(default=list, blank=True)
    palette_fetched_at = models.DateTimeField(null=True, blank=True)

class Track(models.Model):
    spotify_id = models.CharField(max_length=64, primary_key=True)
    album = models.ForeignKey(Album, on_delete=models.CASCADE, related_name='tracks')
    track_number = models.PositiveSmallIntegerField(default=1)
    assigned_color_hex = models.CharField(max_length=7, blank=True)
    artists = models.ManyToManyField(Artist, related_name='tracks', blank=True)
```

Spotify IDs are used as primary keys directly. This means:
- No surrogate integer key that could conflict on import
- `get_or_create(spotify_id=...)` is the natural upsert pattern
- The ID is meaningful — it can be used directly in Spotify API calls

`Track.assigned_color_hex` is set after the album palette is extracted:
```
assigned_color_hex = palette[track_number % len(palette)]
```
A 12-track album with 8 palette colors would assign colors: 0,1,2,3,4,5,6,7,0,1,2,3.
The wrap-around means no two adjacent tracks share a color, and the whole album stays within its visual palette family.

---

### Runs and GPS data: Run → RunPoint

```python
class Run(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='runs')
    source = models.CharField(max_length=16, choices=SOURCE_CHOICES)  # nrc / fit / strava
    external_id = models.CharField(max_length=256)
    # ... summary stats ...

    class Meta:
        unique_together = [('user', 'source', 'external_id')]

class RunPoint(models.Model):
    run = models.ForeignKey(Run, on_delete=models.CASCADE, related_name='points')
    timestamp = models.DateTimeField()
    distance_m = models.FloatField()
    lat = models.FloatField(null=True, blank=True)
    lon = models.FloatField(null=True, blank=True)
    # ...

    class Meta:
        indexes = [models.Index(fields=['run', 'timestamp'])]
```

`Run.id` is a UUID (not an integer) so that when the database moves to Supabase, there's no risk of integer ID collisions between environments or users.

`unique_together = [('user', 'source', 'external_id')]` is the idempotency key. For a `.fit` file, `external_id` is the filename. For an NRC JSON, it's the activity UUID. Running the import twice against the same files will not create duplicate runs.

`RunPoint` uses an integer PK because it's high-volume (~1,000 rows per run, ~233 runs = ~230K rows per user) and never needs a stable external identifier. The composite index on `(run_id, timestamp)` makes route queries fast.

---

### Listening history: PlayEvent

```python
class PlayEvent(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='play_events')
    track = models.ForeignKey(Track, on_delete=models.CASCADE, related_name='play_events')
    played_at = models.DateTimeField(db_index=True)
    source_file = models.CharField(max_length=512, blank=True)

    class Meta:
        unique_together = [('user', 'played_at', 'track')]
```

The 447 Spotify CSV files overlap in time — each file is an export window, and consecutive windows share timestamps at the edges. `unique_together` on `(user, played_at, track)` silently discards duplicates during import without requiring pre-processing of the files.

`source_file` records which CSV the event came from — useful for debugging import issues and for re-importing a specific date range.

---

### The join table: RunTrackMatch

```python
class RunTrackMatch(models.Model):
    run = models.ForeignKey(Run, on_delete=models.CASCADE, related_name='track_matches')
    play_event = models.ForeignKey(PlayEvent, on_delete=models.CASCADE, related_name='run_matches')
    distance_at_start_m = models.FloatField()
    percent_run = models.FloatField()  # 0.0 to 1.0
```

This table is the output of the time-based join (Step 3 of the import pipeline). It is derived data — it could be recomputed from `Run` and `PlayEvent` — but storing it makes the frontend fast. A route map query only needs:

```sql
SELECT rp.lat, rp.lon, t.assigned_color_hex
FROM core_runpoint rp
JOIN core_runtrackmatch rtm ON rtm.run_id = rp.run_id
    AND rtm.percent_run <= (rp.distance_m / run.distance_m)
JOIN core_playevent pe ON pe.id = rtm.play_event_id
JOIN core_track t ON t.spotify_id = pe.track_id
WHERE rp.run_id = '<uuid>'
ORDER BY rp.timestamp;
```

No re-running the merge algorithm at request time.

---

## Entity relationship summary

```
auth_user (Django built-in)
    │
    ├──< Run ──────────────────< RunPoint
    │    │
    │    └──< RunTrackMatch >──┐
    │                          │
    └──< PlayEvent >───────────┘
              │
              ▼
           Track >──< Artist   (M2M: core_track_artists)
              │
              ▼
           Album
         (palette_json → assigned_color_hex on Track)
```

---

## Design rationale

**Why not store color on PlayEvent or RunTrackMatch?** Color is a property of the track (specifically its album). Storing it on the track means updating it in one place when album colors are extracted, rather than backfilling millions of play event rows.

**Why JSON field for `palette_json`?** The palette is a small ordered list (≤8 items) that is always read and written as a whole. It doesn't need to be queried element-by-element. A JSON field is simpler than a related table.

**Why nullable sensor fields everywhere?** Different sources have different data quality. Garmin files sometimes have `heart_rate = 0` (watch worn without HR strap). NRC files may not have GPS. Rather than rejecting incomplete data, all sensor fields are nullable and stored as-is.

---

## Learning takeaway

The schema has one structural insight: split data into what's **shared** (music catalog) and what's **owned** (activity data). The join between them (`RunTrackMatch`) is pre-computed and stored so queries stay simple. UUID primary keys on user-owned records make the eventual Supabase migration straightforward.

---

*Did this section make sense?*
*[ Got it ] [ Needs more detail ] [ Still confused ]*
*Record feedback in `meta/comprehension_signals.json`*
