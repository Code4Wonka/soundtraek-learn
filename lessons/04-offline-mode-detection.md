# Offline Mode Detection & Interpolation

## The Problem

During trail runs in areas with poor cell signal (mountains, canyons), Spotify loses connectivity and enters offline mode. When the app reconnects to the network, Spotify batch-logs all the songs that played during the offline period. This creates two problems:

1. **Compressed timestamps** — All offline plays get logged with nearly identical timestamps (within 1-6 seconds of each other), instead of spread across the actual time they were played.
2. **Missing GPS correlation** — Because the timestamps are batched and not real, the song-to-location matching algorithm (`match_runs.py`) can't accurately map these plays to where in the run they occurred.

**Example from RunSedona Half Marathon:**
- Strava connection lost at 9:05 (race clock: ~3.5 miles)
- Spotify reconnected at 9:29 (race clock: ~7.5 miles)
- 4 songs played offline during those 24 minutes of running
- All 4 logged with played_at timestamps within 1 second: `09:05:00`, `09:05:01`, `09:05:06`, `09:30:06`
- Chart shows all 4 clustered at one location instead of spread across 4 miles

---

## The Solution: Offline Cluster Detection + Position Interpolation

The solution uses three complementary strategies:

### 1. **Cluster Detection** — Identify offline batches

Offline batches have a distinctive pattern:
- **Large silence gap** before the batch (cell signal lost)
- **Rapid-fire timestamps** within the batch (< 12 seconds apart)
- **Normal gaps** after the batch (signal restored, real-time plays resume)

The detection algorithm (`detect_offline_clusters()`) works by:

1. Iterate through play timestamps in order
2. Find consecutive plays with gap < 12 seconds (batched)
3. Check if this cluster is preceded by a gap > 5 minutes (validates offline period)
4. Mark all plays in the cluster except the **last one** (anchor) as offline
5. The anchor is the last track in the cluster — it was matched correctly because it's closest to when signal was restored

**Why keep the anchor?** The anchor track's timestamp is nearest the reconnection and is correctly matched to GPS data. We use it as a temporal reference point to calculate backward.

### 2. **Position Interpolation** — Find where offline plays actually occurred

Since we know the exact song durations (from Spotify), we can walk backward from the anchor's known position.

Algorithm (`interpolate_offline_track_position()`):

1. Start at anchor's timestamp and GPS distance (both known from normal matching)
2. For each offline track working backward:
   - Subtract the track's duration from the timestamp
   - Interpolate the GPS distance at that earlier time using RunPoint data
3. The result: estimated distance_m where each offline track started

**Linear interpolation detail:**
```
RunPoints are sampled every 1-2 seconds with (timestamp, distance_m).
To find distance at any timestamp:
  - Find the two adjacent RunPoints [t1, d1] and [t2, d2]
  - If target time is between them, interpolate:
    distance = d1 + (target_time - t1) / (t2 - t1) * (d2 - d1)
```

### 3. **Visualization** — Render offline tracks distinctly

Offline tracks are rendered on the chart with visual indicators:
- **Dashed border** (not solid) — shows they're estimated, not measured
- **Blue-gray color** — distinct from real-time colored tracks
- **Chart label suffix** — `"(offline est.)"` annotation

In track stat cards, offline plays show:
- WiFi-off icon 🔌 (or lucide `WifiOff`)
- "Estimated" note in hover tooltip

---

## Code Walkthrough

### Backend: `core/views.py`

#### `detect_offline_clusters(match_list)` — ~55 lines

**Input:** List of RunTrackMatch objects, sorted by distance

**Returns:** Dictionary mapping `play_event_id` → `True` if offline-batched

**Key variables:**
- `OFFLINE_CLUSTER_GAP_S = 12` — threshold for "batched" timestamps
- `MIN_GAP_BEFORE_CLUSTER_S = 300` — minimum silence before cluster (5 min)
- `offline_map` — output dict, initialized empty

**Logic:**
```python
for i in range(1, len(match_list)):
    gap = (m.play_event.played_at - prev.play_event.played_at).total_seconds()
    if gap < 12:  # clustered
        # walk back to find cluster start (j)
        # check pre_gap >= 300
        # find cluster end (k)
        # mark [j, k) as offline
```

**Edge cases handled:**
- Single-track runs (no clusters possible)
- Cluster at beginning of run (j == 0, pre_gap = inf)
- Overlapping clusters (handled by sequential iteration)

---

#### `interpolate_offline_track_position()` — ~50 lines

**Input:**
- `offline_id` — play_event_id of offline track
- `match_list` — all tracks in run
- `run` — Run object
- `offline_map` — output from detect_offline_clusters()

**Returns:** float distance_m (estimated), or None if interpolation fails

**Key steps:**

1. **Find indices:**
   ```python
   offline_idx = index in match_list where play_event_id matches
   anchor_idx = first non-offline track after offline_idx
   ```

2. **Build lookup table:**
   ```python
   point_times = [(timestamp, distance_m), ...]  from all RunPoints
   ```

3. **Define interpolation function:**
   ```python
   def distance_at_time(dt):
       # find dt in point_times
       # linear interpolation between adjacent points
       return distance_m at dt
   ```

4. **Walk backward from anchor:**
   ```python
   estimated_time = anchor.play_event.played_at
   for each track between anchor and offline_track:
       estimated_time -= track.duration_ms / 1000
   return distance_at_time(estimated_time)
   ```

**Error handling:**
- Returns None if offline_id not found
- Returns None if anchor_idx >= len(match_list)
- Returns None if no RunPoints (can't interpolate)

---

### Frontend: `run-pace-chart.tsx`

#### Type updates

```tsx
interface OfflineTrack {
  track_name: string
  distance_mi: number
  duration_s: number | null
  avg_pace: number | null
  art_url: string
  artists: string[]
}

// Added to RunDetailData:
offline_tracks: OfflineTrack[]
```

#### Chart rendering

In the `datasets` useMemo:

```tsx
for (const offlineTrack of data.offline_tracks || []) {
  // For each offline track, create a synthetic dataset
  // Points within ±0.5 miles of estimated distance are included
  // Dataset is rendered with dashed border
  datasets.push({
    label: `${offlineTrack.track_name} (offline est.)`,
    data: offlineData,
    borderDash: [4, 4],  // dashed appearance
    borderColor: "rgba(100,120,150,0.8)",  // blue-gray
    spanGaps: true,  // allow breaks in data
  })
}
```

---

## API Response Changes

`GET /api/runs/{id}/chart/` now returns:

```json
{
  "records": [...],
  "tracks": [...],
  "track_stats": {...},
  "run_pace_min_mile": 11.25,
  "offline_tracks": [
    {
      "track_name": "Dear, Snow",
      "distance_mi": 4.95,
      "duration_s": 180,
      "avg_pace": 11.2,
      "art_url": "...",
      "artists": ["Artist 1", "Artist 2"]
    },
    ...
  ]
}
```

---

## Example: RunSedona Half Marathon

**Data:**
- Total distance: 13.11 mi
- Total duration: 149.6 min (11:25/mi average)
- Spotify gap: ~24 minutes (9:05–9:29, race time)

**Offline cluster detected:**
```
[9:05:00] Dear, Snow          gap=764s (12.7 min)  ← pre-gap (offline signal lost)
[9:05:01] Rap Niggas          gap=1s               ← clustered (offline batch)
[9:05:06] Ocean Views         gap=6s               ← clustered (ANCHOR: correctly matched)
[9:29:55] Next track          gap=1489s (24.8 min) ← normal gap, signal restored
```

**Interpolation for "Dear, Snow":**
```
Anchor (Ocean Views): played_at = 09:05:06, distance = 5.53 mi

Walk backward:
  09:05:06 - (230s / track) = 09:00:56  → distance ≈ 5.23 mi (from RunPoint lookup)
  09:00:56 - (226s / Rap Niggas) = 08:56:10 → distance ≈ 4.95 mi
  (interpolated position for Dear, Snow = 4.95 mi)

Chart renders:
  - Blue-gray dashed line from 4.95 to ~5.1 mi
  - Label: "Dear, Snow (offline est.)"
```

---

## Design Rationale

**Why this approach?**

1. **No ground truth for offline times** — We can't reconstruct exact play times (that data doesn't exist). Interpolation is an educated guess, not perfect.

2. **Anchor as reference** — The last track in a cluster is closest to reconnection time, so its timestamp is most reliable. Using it as the anchor minimizes error propagation.

3. **Duration-based backward walk** — Spotify durations are exact and immutable. Walking backward from anchor ensures positions are internally consistent and respect actual song lengths.

4. **Visual distinction (dashed)** — Dashed lines signal "estimated, use with caution" — important for users to know these aren't measured facts.

5. **Distance tolerance (±0.5 mi)** — RunPoints are discrete (every 1-2 sec), so we use a small window around estimated position. Too tight and we miss data; too wide and we overplot.

---

## Limitations & Future Improvements

**Current limitations:**
- Assumes uniform running pace during offline period (not true for aid stations, hills)
- Sensitive to large position jumps in RunPoint data (e.g., GPS glitches)
- Does not account for pauses/stops during offline period

**Future improvements:**
- Use avg_pace from RunTrackMatch to calculate more accurate distance
- Flag interpolations with high confidence vs low confidence
- Allow user to manually adjust offline track positions in the UI
- Detect and filter out GPS outliers before interpolation

---

## Evidence

- **Backend implementation:** `core/views.py` lines 128–160 (detection), 161–191 (interpolation)
- **API endpoint:** `GET /api/runs/{id}/chart/` includes `offline_tracks` field
- **Frontend rendering:** `run-pace-chart.tsx` lines 49–56 (types), 317–340 (dataset building)
- **Test data:** RunSedona Half Marathon (Feb 7, 2026, trail run in AZ mountains)

---

## Learning Takeaway

**This pattern solves a real mobile/outdoor problem** — Apps that log events in batches when connectivity resumes are common (Strava, Fitbit, etc.). Being able to detect and interpolate these batches from available ground-truth data (GPS, durations) is a useful technique for data recovery and enrichment in any sensor-based app.

The key insight: **You can recover missing timestamps using auxiliary data** (song durations, GPS timestamps) and work backward from the most reliable reference point (anchor track).
