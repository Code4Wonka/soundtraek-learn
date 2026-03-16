# Lesson 05: Offline Mode Detection

## What This Lesson Covers

When Spotify loses network connectivity while you're running, it can't log what you're listening to in real-time. Instead, when you reconnect, Spotify batches all the offline plays at nearly identical timestamps. This lesson explains:

1. **The offline problem** — what happens to Spotify data when you lose signal
2. **The detection algorithm** — how we identify batched plays in the timestamp stream
3. **The clustering insight** — why the last track in a cluster is special (the "anchor")
4. **Gating the visualization** — how offline detection gates the chart rendering strategy

---

## The Offline Problem

### What Is Offline Mode?

When you lose network signal during a run:
- Spotify can't communicate with its servers
- It can't log `played_at` timestamps in real-time
- Your device queues up the tracks you played

When you reconnect:
- Spotify sends all offline plays to the server at once
- They all get assigned nearly identical `played_at` timestamps (batched)
- The Spotify API returns them clustered together

### Why This Breaks Chart Visualization

In a normal run, each song has an accurate `played_at` timestamp. We can match it to GPS distance by looking up what distance you were at when that timestamp occurred.

In an offline cluster, all songs have timestamps like:
```
Song A: 2026-03-15 14:23:47.123 UTC
Song B: 2026-03-15 14:23:48.005 UTC
Song C: 2026-03-15 14:23:48.892 UTC
```

All within 1 second of each other. This is physically impossible—you couldn't have played three different songs in one second. These timestamps don't represent *when* the songs played, just *when they were logged*.

### Example: RunSedona Half Marathon

On a recent half-marathon route with spotty cell coverage:
- **Miles 2.4–7.5**: Lost signal for ~5 miles
- **At reconnection (mile 7.5)**: Spotify logged 4 songs with timestamps separated by milliseconds
- **Problem**: All 4 get mapped to roughly the same GPS distance, visually stacking them at one point instead of spreading them across the 5-mile gap

---

## The Detection Algorithm

### High-Level Strategy

We need to identify which plays are "offline-batched" and which are real. The key insight: **the last track in an offline cluster is the anchor**—it was playing when you reconnected and got logged at the correct time. All earlier tracks in the cluster are the offline batches.

### Thresholds

```python
OFFLINE_CLUSTER_GAP_S = 12      # consecutive plays within 12s = potentially clustered
MIN_GAP_BEFORE_CLUSTER_S = 300  # silence > 5 min before cluster = likely offline
```

Why these values?
- **12 seconds**: A running playlist rarely has songs with <12s gaps between plays in normal conditions. Fast transitions (user skipping, shuffle) happen, but not within such tight windows. Any cluster with <12s between plays is suspicious.
- **300 seconds (5 min)**: After 5+ minutes of no music logged, a sudden burst of plays is very likely offline batching, not normal listening.

### The Algorithm

```
for each song in the run's song sequence:
  if (current_song.played_at - previous_song.played_at) < 12 seconds:
    # Found a clustered pair—might be offline
    # Walk backward to find cluster start
    cluster_start = i
    while cluster_start > 0 and gap_before < 12s:
      cluster_start -= 1

    # Check if cluster is preceded by long silence
    silence_before_cluster = cluster_start.played_at - song_before_cluster.played_at
    if silence_before_cluster > 5 minutes:
      # Confirmed: this is an offline cluster
      # Find cluster end
      cluster_end = i
      while cluster_end + 1 < length and gap_after < 12s:
        cluster_end += 1

      # Mark all songs except last (anchor) as offline
      for song in cluster[cluster_start : cluster_end - 1]:
        mark_as_offline(song.id)

  move to next song
```

### Code Implementation

In [core/views.py:128-191](core/views.py#L128-L191), the function `detect_offline_clusters()`:

```python
def detect_offline_clusters(match_list):
    """
    Detect Spotify plays that were batched due to offline mode.

    When Spotify reconnects after losing signal, it logs all offline plays with
    nearly identical timestamps. This function identifies such clusters and marks
    which play_event_ids are "offline-batched" (not the anchor).

    Algorithm:
    1. Find clusters: consecutive plays with gap < 12 seconds
    2. Check cluster is preceded by large silence (> 5 min)
    3. Mark all but last track in cluster as offline (last is anchor, was correctly matched)

    Returns:
        dict mapping play_event_id → True if offline-batched
    """
    OFFLINE_CLUSTER_GAP_S = 12
    MIN_GAP_BEFORE_CLUSTER_S = 300

    offline_map = {}  # play_event_id → is_offline

    if len(match_list) < 2:
        return offline_map

    i = 1
    while i < len(match_list):
        m = match_list[i]
        prev = match_list[i - 1]
        gap = (m.play_event.played_at - prev.play_event.played_at).total_seconds()

        if gap < OFFLINE_CLUSTER_GAP_S:
            # Found a clustered pair — walk back to find cluster start
            j = i
            while j > 0:
                g = (match_list[j].play_event.played_at - match_list[j - 1].play_event.played_at).total_seconds()
                if g >= OFFLINE_CLUSTER_GAP_S:
                    break
                j -= 1

            # j is first member of cluster; check if preceded by large gap
            pre_gap = float('inf')
            if j > 0:
                pre_gap = (match_list[j].play_event.played_at - match_list[j - 1].play_event.played_at).total_seconds()

            if pre_gap >= MIN_GAP_BEFORE_CLUSTER_S:
                # This is a valid offline cluster; find its end
                k = i
                while k + 1 < len(match_list):
                    g = (match_list[k + 1].play_event.played_at - match_list[k].play_event.played_at).total_seconds()
                    if g >= OFFLINE_CLUSTER_GAP_S:
                        break
                    k += 1

                # k = anchor (last in cluster); all others in [j, k) are offline
                for idx in range(j, k):
                    offline_map[match_list[idx].play_event_id] = True

                i = k + 1
            else:
                i += 1
        else:
            i += 1

    return offline_map
```

### Key Insight: The Anchor Track

In a 4-song offline cluster:
```
Song A (offline, batched)
Song B (offline, batched)
Song C (offline, batched)
Song D (anchor)
```

**Song D is the anchor** because:
- It was the last song playing when you reconnected
- Its `played_at` timestamp is accurate (logged at reconnection, not when it started playing)
- It gets mapped to the correct GPS distance in the run
- The GPS distance of Song D becomes the "starting point" for backward interpolation of Songs A, B, C

The detection algorithm marks A, B, C as offline (they go in `offline_map`), but leaves D unmarked so it's rendered as a normal colored segment.

---

## How Offline Detection Gates Rendering Strategy

Offline detection doesn't just identify offline tracks—it's a **prerequisite signal** that determines which chart rendering strategy to apply.

### Two Visualization Modes

#### Mode 1: No Offline Detected (Regular Run)

Endpoint strategy: **Extend to next track's start**

```python
if not has_offline:
    # Clean run: extend each track to the next track's start
    # Result: full coverage, no gray gaps
    end = match_list[i + 1].distance_at_start_m if i + 1 < len(match_list) else float('inf')
```

- **Why**: Without offline mode, song timestamps are accurate and evenly spaced. Extending each song to the next track's start ensures 100% of the run is colored (no gaps).
- **Result**: Solid colored coverage from start to finish, no gray unmatched segments.

#### Mode 2: Offline Detected (Run with Offline Cluster)

Endpoint strategy: **Duration × pace**

```python
if has_offline:
    # Offline run: use song duration to estimate endpoint
    # Offline-batched songs won't occupy unrealistic distances
    if m.duration_s and m.avg_pace_min_mile and m.avg_pace_min_mile > 0:
        miles_covered = (m.duration_s / 60.0) / m.avg_pace_min_mile
        end = start + miles_covered * METERS_PER_MILE
    else:
        # Fallback: next non-offline track
        next_real = next(...)
        end = next_real.distance_at_start_m if next_real else float('inf')
```

- **Why**: When offline tracks are present, we can't trust cluster timestamps. Using each track's actual duration ensures it only occupies the distance it was truly playing.
- **Result**: Accurate track lengths + gray gaps in offline zones + dashed interpolated segments for offline songs.

### The Gating Logic

```python
intervals = []
has_offline = bool(offline_map)  # <-- GATE: prerequisites signal

for i, m in enumerate(match_list):
    if offline_map.get(m.play_event_id, False):
        continue  # offline tracks rendered as dashed; skip here

    start = m.distance_at_start_m

    if has_offline:
        # Offline cluster detected: use duration-based endpoint
        ...
    else:
        # Clean run: extend to next track
        ...

    intervals.append((start, end, m.play_event.track.name))
```

The `has_offline` boolean—derived from the offline detection algorithm—gates the entire interval-building strategy. A single `bool()` call controls two different visualization modes.

---

## Real Example: RunSedona Half Marathon

### The Data

RunSedona is a 13.1-mile half-marathon with an offline cluster at miles 2.4–7.5 (spotty cell coverage on a scenic road).

### Offline Detection Output

```
detected offline_map = {
    'pe_offline_1': True,    # Song 1 (batched)
    'pe_offline_2': True,    # Song 2 (batched)
    'pe_offline_3': True,    # Song 3 (batched)
    'pe_offline_4': False,   # Song 4 (anchor, not in map)
}

bool(offline_map) = True    # Gate enabled
```

### Chart Rendering

**Before offline zone (miles 0–2.4):**
- Regular tracks with accurate timestamps
- Interval strategy: extend to next track's start
- Result: solid colored segments

**Offline zone (miles 2.4–7.5):**
- Real tracks using duration × pace endpoint
- Anchor track (Song 4) colored, accurate length
- Offline tracks (Songs 1–3) rendered as dashed lines
- GPS points with no song assigned: gray (unmatched)
- Result: accurate track lengths, gray gaps, dashed offline lines

**After offline zone (miles 7.5–13.1):**
- Regular tracks resume
- Interval strategy: extend to next track's start
- Result: solid colored segments

### Final Chart Metrics

- 3443 colored points (matched to regular + anchor tracks)
- 5536 gray points (GPS in offline zone with no matched track)
- 7 dashed segments (interpolated positions of offline tracks 1–3)
- 0 songs spanning unrealistic distances
- Visualization accurately represents the offline cluster and its impact on the run

---

## Design Rationale

### Why Threshold These Values?

**12-second cluster gap:**
- Spotify's typical shuffle and skip behavior rarely produces <12s gaps in normal listening
- Clustering happens with millisecond spacing
- 12s is a conservative boundary that catches real clusters without false positives

**5-minute silence before cluster:**
- Normal running doesn't have 5-minute music gaps (you'd stop and turn it off)
- This threshold reliably separates offline batches from real silence/pause gaps

### Why the Anchor Approach?

The last track in a cluster is trustworthy because:
- It was playing when reconnection happened
- Its `played_at` timestamp reflects server receipt time (at reconnection), making it mappable to GPS distance
- It anchors the interpolation of earlier offline tracks

### Why Not Interpolate All Offline Tracks?

Offline-batched timestamps are fundamentally unreliable—they're all ~0ms apart, not spread across the cluster's time window. We could artificially spread them, but:
- Artificial spreading is speculative, not grounded in real data
- The dashed lines on the chart make it clear these positions are estimates, not GPS-logged
- The anchor track gives us one reliable reference point to work backward from

---

## Learning Takeaway

Offline detection is a prerequisite signal that gates chart rendering strategy. By detecting which plays are batched vs. real, we can choose the right visualization for each run type:

- **No offline**: Full coverage, no gaps (songs extend to next track)
- **Has offline**: Accurate track lengths, gray gaps, dashed interpolations (songs use duration × pace)

The algorithm is simple but powerful: find clusters (tight timestamps) preceded by long silence, mark all but the anchor as offline, then let that single boolean control the entire rendering pipeline.

This design avoids regressions because each visualization mode gets its own strategy, chosen based on the actual data in the run.

---

## Evidence

- `core/views.py:128-191` — `detect_offline_clusters()` implementation
- `core/views.py:260-347` — `build_offline_segments` implementation with backward walk reconstruction
- `core/views.py:377-404` — conditional interval-building logic using `has_offline` gate
- `frontend/src/components/run-pace-chart.tsx` — chart rendering with offline track handling

---

## Next Steps

Once you understand offline detection, read:
- [Lesson 04: Song-Run Matching](04-song-run-matching.md) — covers timestamp deduction, clamping, and backward walk reconstruction
- [Architecture Overview](../architecture.html) — data flow from import to visualization
