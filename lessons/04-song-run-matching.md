# Lesson 04 — The Mystery of the Spotify Timestamp

## The Problem in 30 Seconds

Spotify's export tells you when a song **ended**, not when it started. Imagine getting a crime scene report that says "the victim died at 3 PM" — you still need to figure out where they were at 2 PM. SoundTræk solves this puzzle with three tricks: **math, heuristics, and backward walking**. The result: your running map colored by the songs you actually heard.

---

## Act 1: The One-Sided Clock

Spotify records a simple fact: `played_at = 15:05:00` (the song finished at 3:05). But it never records the start time. So we deduce it:

```
If song duration is 3 minutes:
  Song ended at 15:05:00
  So it started at 15:02:00
```

This works **usually**. But when you skip a song after 10 seconds, the math breaks:

| Song | You heard it from | Ended at | Deduced start | Problem? |
|---|---|---|---|---|
| Intimidated | 21:39:20 – 21:42:47 | 21:42:47 | 21:39:20 | ✓ Correct |
| All of You | 21:42:47 onwards | 21:45:44 | **21:42:11** | ❌ 36s too early! |

The deduced start is 36 seconds **before the last song ended**. When we ask "what was playing at 21:42:16?", the algorithm sees "All of You started at 21:42:11 (more recent) vs. Intimidated at 21:39:20" and picks the wrong song.

### The Fix: Clamping (120-second rule)

If two songs overlap by less than 2 minutes, it's definitely a skip — fix it. If they overlap by 4+ minutes, something else is happening (spoiler: offline batch). So we clamp with a threshold:

```python
overlap = prev_song_end - next_song_deduced_start
if 0 < overlap < 120 seconds:
    clamp the next song's start to the prev song's end
```

After clamping, "All of You" starts at 21:42:47 (when the previous song ended), and merge_asof correctly finds "Intimidated" as the song playing at 21:42:16. ✓

---

## Act 2: The Offline Heist

While you're running with no internet, Spotify caches your plays locally. When you reconnect, the phone syncs **all the buffered plays at once** — reporting them all with the same timestamp (the sync moment), not the actual play times.

RunSedona Half Marathon (2026-02-07) shows this perfectly. At 15:34:49, Spotify syncs 7 songs from the previous 27 minutes:

| Song | Duration | Reported played_at | Actual played |
|---|---|---|---|
| Early Bird | 2:39 | 15:34:49.396 | ~15:07:39 – 15:10:18 |
| Million While You Young | 4:04 | 15:34:49.572 | ~15:10:18 – 15:15:22 |
| Dedication | 4:06 | 15:34:49.586 | ~15:15:22 – 15:19:28 |
| Last Time I Checc'd | 3:46 | 15:34:49.803 | ~15:19:28 – 15:23:14 |
| Victory Lap | 3:58 | 15:34:49.814 | ~15:23:14 – 15:27:12 |
| Lit | 4:39 | 15:34:59.545 | ~15:27:12 – 15:31:51 |
| **Right Hand 2 God (anchor)** | 3:08 | 15:34:59.861 | ~15:31:51 – 15:34:59 |

The algorithm detects this pattern:

- A silence of 5+ minutes in the listening history
- Followed by multiple songs with `played_at` within 12 seconds
- That's an offline batch — mark it for special handling

The last song in the batch (**Right Hand 2 God**) is the "anchor" — its timestamp is closest to reality. The others are estimated.

---

## Act 3: Reconstructing the Timeline

To position the other 6 songs, we walk **backward from the anchor**:

```
Right Hand 2 God (3:08) ends at 15:34:59
  Subtract 3:08  ↑
Lit (4:39) plays 15:27:12 – 15:31:51
  Subtract 4:39  ↑
Victory Lap (3:58) plays 15:23:14 – 15:27:12
  ... and so on back to Early Bird
```

From a 10-second timestamp, we reconstruct the entire 27-minute playlist. That's the power of the backward walk.

### Code: The backward walk

```python
cluster.sort(key=lambda m: m.play_event.played_at)  # Sort by actual play order

# Walk backward from anchor
t = anchor_time
track_times = []
for i in range(len(cluster) - 1, -1, -1):
    t = t - timedelta(seconds=cluster[i].play_event.track.duration_ms / 1000.0)
    track_times.append((cluster[i], t))

track_times.reverse()  # Forward order: oldest first
```

The sort by `play_event.played_at` is crucial — offline songs collapsed to the same time have arbitrary distance order, but their played_at tells the truth about which was first.

---

## Why This Matters

Without these fixes:

- **Regular runs** show the wrong song at the start (user skipped once → algorithm picks wrong track for first 30 seconds)
- **Offline runs** show 70% gray (unmatched) instead of the colorful route you remember

With them:

- Every run is accurately matched
- The route becomes a visual memory of your playlist
- Even when your phone went offline, the algorithm recovers the songs you heard

---

## The Big Picture

This lesson illustrates a broader principle: **data is always incomplete, and the art of system design is making smart assumptions**. Spotify's incomplete timestamps aren't a bug — they're a constraint. SoundTræk treats the constraint as a puzzle and solves it with three layers:

1. **Math** — deduce start times from end times (imperfect but necessary)
2. **Heuristics** — detect offline batches by looking for patterns
3. **Reconstruction** — walk backward to assign realistic time windows

Each layer catches where the previous one fails. Together, they turn a partial signal into a complete story.

---

## Evidence

- [match_runs.py](../../core/management/commands/match_runs.py): Clamping logic (lines 96–101)
- [views.py](../../core/views.py): `detect_offline_clusters` (lines 126–189), `build_offline_segments` (lines 260–347)
- [run-pace-chart.tsx](../../frontend/src/components/run-pace-chart.tsx): Visualization (lines 446–476)
