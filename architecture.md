# SoundTræk — Architecture

## System layers

```
┌─────────────────────────────────────────────────────────┐
│  Data Sources (external, read-only)                     │
│  Spotify CSVs · NRC JSONs · Garmin .fit · Strava CSV    │
└───────────────────────┬─────────────────────────────────┘
                        │ management commands
┌───────────────────────▼─────────────────────────────────┐
│  Import Pipeline                                         │
│  import_spotify · import_runs · match_runs               │
│  extract_album_colors                                    │
└───────────────────────┬─────────────────────────────────┘
                        │ Django ORM
┌───────────────────────▼─────────────────────────────────┐
│  Database (SQLite → Supabase)                            │
│  Album · Artist · Track                                  │
│  Run · RunPoint · PlayEvent · RunTrackMatch              │
└───────────────────────┬─────────────────────────────────┘
                        │ Django views
┌───────────────────────▼─────────────────────────────────┐
│  Frontend (not yet built)                                │
│  Route map · Run stats · Music stats                     │
└─────────────────────────────────────────────────────────┘
```

## Key architectural decisions

| Decision | Choice | Reason |
|---|---|---|
| Framework | Django 4.2 | Batteries-included ORM, admin, auth |
| DB (local) | SQLite | Zero setup, fast iteration |
| DB (production) | Supabase (Postgres) | Managed Postgres, RLS for multi-user |
| PK strategy | UUID for user records, Spotify ID for catalog | UUID = no migration collision; Spotify ID = natural dedup |
| Color storage | Per-album palette JSON + per-track hex | Extract once, reuse forever; deterministic assignment |
| Import strategy | Idempotent management commands | Safe to re-run; data files stay outside repo |
| Deployment | Vercel + GitHub Actions | Push-to-deploy for solo development |

## Data volume estimates (per user at beta scale)

| Table | Rows |
|---|---|
| PlayEvent | ~20,000 |
| Run | ~300–500 |
| RunPoint | ~300,000–500,000 |
| RunTrackMatch | ~3,000–8,000 |
| Track / Album / Artist | Shared — ~50,000 total |
