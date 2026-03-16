# Lesson 06 — The Music Page

How 447 CSV files became a searchable, interactive listening history — and what the numbers actually tell you.

## What This Page Does

The Music tab is the primary lens into your listening history. Every Spotify play event linked to a run becomes a queryable row. The page answers three questions:
- **What have I listened to?** The searchable track list.
- **How often?** Play count, run count, total distance and time.
- **How did my pace change?** The pace chart shows improvement (or decline) over time for a given track.

Think of it as your Spotify Wrapped, but interactive and continuously updated as you run and listen.

## The Track List: Search, Sort, Pagination

The Music tab is a paginated table of 447 unique tracks from your Spotify history. It's built for scanning and filtering, not scrolling endless lists.

### Design Decision: Server-Side Search

The search runs on the server, not the browser:

```javascript
// MusicPanel.tsx
const params = new URLSearchParams({
  q,
  page: String(p),
  page_size: String(PAGE_SIZE),
  sort: s,
})
const resp = await fetch(`/api/music/?${params}`, { credentials: 'include' })
```

The backend filters with a dual-field query:

```python
# core/views.py
Q(track__name__icontains=q) | Q(track__artists__name__icontains=q)
```

**Why server-side?** With 447 tracks and 3000+ play events, client-side search would require downloading and indexing all that data on every page load. Server-side keeps the wire slim and lets the database do the heavy lifting.

**Trade-off:** Search fires on every keystroke with no debounce — each keystroke hits the API. This costs extra requests, but for a SQLite-backed dev server, it's snappy enough. Once you move to Postgres (see Lesson 07), you still get instant results because Postgres is faster than SQLite.

### Three Sort Modes

The table supports three sort fields:

1. **Last Played** (default) — most recent track at the top. The natural "what did I just listen to?" view.
2. **Play Count** — how often you've played each track. Useful for finding your most-played bangers.
3. **Track Name** — alphabetical. Rarely used, but here for completeness.

Clicking a column header toggles the sort and resets pagination to page 1.

### Pagination: 50 Per Page

The page size is fixed at 50. Not configurable—just right: fast to load, small enough to scan without scrolling forever, large enough that you're not clicking "Next" constantly.

The pagination control shows only when there's more than one page:
```
[First] [Prev] Page 3 of 14 [Next] [Last]
```

### The N+1 Problem (and How It's Solved)

The original naive query was:

```python
# Bad: N+1 queries
for track_id in track_ids:
    artists = Track.objects.get(id=track_id).artists.all()
```

This fetches a track, then fetches its artists, then the next track, then its artists... 50 queries to fetch 50 tracks' artists.

The fix: batch-fetch all artists at once, *after* the aggregation:

```python
# Good: 2 queries total
tracks = PlayEvent.objects.filter(user=user) \
    .values('track_id', 'track__name', 'track__album__art_url') \
    .annotate(play_count=Count('id'), last_played=Max('played_at')) \
    .distinct()

track_ids = [t['track_id'] for t in tracks]
tracks_with_artists = Track.objects.filter(spotify_id__in=track_ids) \
    .prefetch_related('artists')  # Batch fetch via JOIN
```

The `prefetch_related('artists')` tells Django to fetch all artists for all 50 tracks in one query (not 50 queries).

### Empty States That Tell a Story

Two different empty states:

1. **No data imported yet:** "No music plays yet. Import Spotify data to get started."
   - Appears when `play_count === 0` across all tracks.
   - Tells the user they're at the start of a journey.

2. **Search returned nothing:** "No tracks match your search."
   - Appears when a query is entered but returns zero results.
   - Tells the user to try a different query, not that the app is broken.

## The Song Detail Modal: What's Inside and Why

Click a track in the list, and a modal pops up with detailed stats. This is where the design gets interesting—it's not just a data dump, it's a story about *you and this track*.

### Header: Familiar Music-App Design

```
[Album Art: 88×88px] Track Name
                     Artist Name
                     Album Name
                     3:45  ·  Last run: Baylands 3mi • Mar 14, 2026
```

The album art is 88×88 pixels—large enough to see the colors and imagery, small enough that it doesn't dominate the modal. (This size is also the "sweet spot" for album artwork in Spotify's own apps.)

The header also shows the last run you listened to this track on, so you can see the context of the most recent listen.

### Stats Grid: Four Numbers That Matter

Four stat cards, arranged in a single row:

| Plays | Runs | Miles Run | Time Running |
|---|---|---|---|
| Total play events across all runs | Number of distinct runs | Total distance covered (miles) | Total time spent running to this track |
| 47 | 12 | 18.5 | 2h 45m |

**Why four?** Originally this was a 2×3 grid with six cards: Plays, Runs, Miles, Time, *Avg Pace*, and *Best Pace*.

But then came the pace chart (see next section), which encodes both Avg and Best Pace visually with *way* more information—not just the numbers, but the trend, the spread, and the outliers. Duplicating that data as text cards was redundant.

**Design principle:** Don't show the same information two ways unless one way offers genuinely different insights.

### Pace Chart: The Headline Visualization

This is where the modal shines. The pace chart is a **single-line visualization of your pace over time** while listening to this track across all your runs.

**What you're looking at:**

```
Faster ├─────────────●───★───●──────┤
       │      •      |       •      │
Slower └──────────────────────────────┘
       jan          feb      mar
```

- **X-axis:** Time (chronological). If there's a 3-month gap between runs, it *looks* like a gap.
- **Y-axis:** Pace (minutes per mile). "Faster" at the top, "Slower" at the bottom.
- **Blue dots:** Each run you've done to this track. Close together vertically = consistent pace. Spread out = variable pace.
- **Blue dashed line:** Your average pace for this track.
- **Gold star (★):** Your best (fastest) pace for this track. The one moment that counts.
- **Hollow circles:** Outliers that don't fit the pattern (like a recovery walk). They're visible but don't distort the scale.

**Design inspiration:** Strava's "Fitness & Freshness" trend chart, simplified to a single metric and one track. The idea is to answer: *Am I getting faster or slower with this track?*

**Implementation details:**

The y-axis scale uses **IQR clamping**. Here's why:

Imagine you ran 12 times to this track: 10 runs at 8:30/mi, and 1 recovery walk at 11:30/mi (outlier). A naive chart would scale the y-axis from 8:30 to 11:30, compressing the 10 "real" runs into a tiny band at the top.

IQR clamping (Tukey fence) sets the scale to Q1 − 1.5×IQR to Q3 + 1.5×IQR, where Q1 and Q3 are the 25th and 75th percentiles. The outlier still shows as a hollow circle pinned to the edge, but it doesn't mess up the scale for everyone else.

The x-axis uses **proportional time**. The math:
```
x_pixel = (date - min_date) / (max_date - min_date) × chart_width
```

If you have runs on Jan 1, Feb 15, and Mar 30, those gaps are represented proportionally. A 3-month gap looks 3 months wide.

### Run Position Heatmap: When Does This Song Play in Your Runs?

Below the pace chart sits a heatmap:

```
[░░░█████░░░░████░░░░]
Early        Mid        Finish
```

This heatmap bins your runs into 20 segments from start to finish. Each bar is colored by opacity—more opaque = this song plays there more often.

**Design inspiration:** GitHub contribution calendar heatmaps. The visual metaphor is instantly familiar to anyone who uses GitHub: heat = frequency.

**Color choice:** Blue (`#4A6FD4`), not gold. The pace chart already uses gold (for the best pace point), so the heatmap stays in the app's primary blue to avoid color overload.

**What it tells you:** Is this track your warm-up song? Your finishing kick? Do you save it for the second half?

### AI Analysis: Collapsed by Default

At the bottom sits a collapsed accordion labeled "✦ AI Analysis." Click to expand and see a pre-written Claude prompt:

```
I'm analyzing the running performance data for a specific song and want your insights.

Track: "Status Symbol 2" by Buddy (from "Status Symbol")
Song duration: 3:45

My running data across 12 runs:
• Average pace while playing: 8:18/mi
• Best pace achieved: 7:45/mi
• Average elevation change per occurrence: +42 ft
• Total distance covered with this track: 18.5 mi
• Total time I've run to this track: 2h 45m
• Run position: appears mid in my runs
  (early: 28%, mid: 42%, late: 30%)

Please analyze:
1. Is my pace better or worse than typical while this track plays? What does that suggest?
2. What does the elevation pattern tell you about when I'm listening to this track?
3. Given it tends to appear mid in my runs, what does this tell you about how I'm using it psychologically?
4. How should I position this track in future runs for the best performance outcome?
```

Two copy buttons: "Copy Prompt" and "Copy Prompt + Data."

**Design principle:** The AI doesn't do the analysis for you (yet). Instead, it gives you a well-structured prompt you can paste into Claude directly. This keeps the modal lightweight and lets *you* stay in control of the conversation.

The "Copy Prompt + Data" button includes the raw stats as JSON, so Claude can do quantitative analysis if you want.

## Performance Choices

### Why Count Matters (M2M Join Duplication)

The play events table has a many-to-many relationship with artists (one play event can involve multiple artists, though usually one). If you naively aggregate without `distinct()`:

```python
# Bad: play_count inflates
PlayEvent.objects.filter(user=user) \
    .values('track_id') \
    .annotate(play_count=Count('id'))  # Over-counts if .artists is present
```

With `.distinct()`, Django knows to count each play event only once, even if the M2M join creates duplicate rows:

```python
# Good: correct count
PlayEvent.objects.filter(user=user) \
    .values('track_id') \
    .annotate(play_count=Count('id')) \
    .distinct()
```

### Single Aggregate Call for Stats

In `api_track_detail`, one `.aggregate()` call fetches all the stats at once instead of multiple queries:

```python
matches_qs.aggregate(
    run_count=Count('run_id', distinct=True),
    avg_pace_min_mile=Avg('avg_pace_min_mile'),
    best_pace_min_mile=Min('avg_pace_min_mile'),
    total_distance_mi=Sum(...),  # Computed field via ExpressionWrapper
    total_run_time_s=Sum('run__duration_s'),
)
```

This is 1 query, not 5.

### Heatmap: Counter + Normalization

Building the 20-bin heatmap:

```python
from collections import Counter
percent_runs = [m.percent_run for m in matches_qs]
bins = Counter(min(19, int(p * 20)) for p in percent_runs)
max_count = max(bins.values())
heatmap = [(bins.get(i, 0) / max_count) * 100 for i in range(20)]
```

This is O(n) with one pass through the data. Each bar is then normalized to 0–100 so the opacity scale is consistent.

### IQR Clamping Rationale

Why the Tukey fence (Q1 − 1.5×IQR to Q3 + 1.5×IQR) for the pace chart?

It's the standard outlier detection method in statistics. It's also used in box plots, which is why it feels visually familiar. The 1.5× multiplier is empirically tuned to flag ~0.7% of normally distributed data as outliers—a good balance between protection and sensitivity.

## Evidence

The Music page is built across three main files:

1. **[frontend/src/components/MusicPanel.tsx](../../frontend/src/components/MusicPanel.tsx)** — The track list component. Handles search, sort, pagination, and the row click that opens the modal.

2. **[frontend/src/components/SongDetailModal.tsx](../../frontend/src/components/SongDetailModal.tsx)** — The modal component. Fetches track details, renders the stats grid, pace chart (via `PaceTimeline` SVG component), heatmap, and AI analysis section.

3. **[core/views.py](../../core/views.py)** — Two API endpoints:
   - `api_music` (GET `/api/music/?q=&sort=&page=`) — returns paginated, searchable track list
   - `api_track_detail` (GET `/api/music/{track_id}/`) — returns full stats, heatmap, pace history, and last run

## Next Steps

Once you're running this in production with a real database (see Lesson 07), consider these optimizations:

- Add full-text search (`SearchVectorField`) for faster fuzzy track/artist matching.
- Cache the pace history and stats for a day to avoid recalculating on every modal open.
- Add filters by date range, BPM, energy level (pulled from Spotify's audio features API).
- Let users save favorite tracks and create playlists within the app.

But for now, the Music page answers its core questions: *What are you listening to, how often, and are you getting faster?* That's enough.
