# Lesson 01 — Project Organization

## What this system does

SoundTræk is a Django web application that answers one question:
**What song was I listening to at each moment of a run — and what did that look like on a map?**

The project takes historical data from two sources (running activity files and Spotify listening exports), joins them by timestamp, and will eventually render a GPS route colored by the album art of whatever was playing.

This lesson covers how the project is organized on disk and why.

---

## Evidence

- `manage.py`
- `soundtraek/settings.py`
- `soundtraek/urls.py`
- `core/views.py`
- `core/models.py`
- `vercel.json`
- `requirements.txt`
- `.github/workflows/ci.yml`
- `assets/wordmark.png`
- `docs/schema.md`

---

## Walkthrough

### Top-level layout

```
SoundTræk/
│
├── soundtraek/          # Django project config (settings, routing, server)
├── core/                # The main application (models, views, logic)
├── tests/               # pytest test suite
├── templates/           # HTML templates (empty — not yet built)
├── static/              # Static assets served by Whitenoise
├── assets/              # Brand assets (wordmark, etc.)
├── docs/                # Schema and architecture documentation
├── learn/               # Project Mirror teaching layer (you are here)
├── audit/               # Project Mirror audit reports
├── meta/                # Project Mirror metadata index
├── .claude/commands/    # Claude Code slash command skills
│
├── manage.py            # Django CLI entry point
├── requirements.txt     # Python dependencies
├── vercel.json          # Vercel deployment routing
├── Procfile             # Gunicorn process definition
├── CLAUDE.md            # Instructions for Claude Code
└── db.sqlite3           # Local SQLite database (not committed)
```

### The two Django layers: `soundtraek/` vs `core/`

Django splits a project into a **project package** and one or more **apps**.

`soundtraek/` is the project package — it holds configuration only:
- `settings.py` — database, installed apps, static files, API keys
- `urls.py` — top-level URL routing (delegates to `core/`)
- `wsgi.py` / `asgi.py` — server entry points for Gunicorn and Vercel

`core/` is the application — it holds everything that actually does work:
- `models.py` — the database schema
- `views.py` — HTTP request handlers
- `urls.py` — URL patterns for the core app
- `migrations/` — auto-generated database migration files

This separation means you could add a second app later (e.g. `api/`) without touching `soundtraek/`.

### How a request flows through the project

```
HTTP Request
↓
soundtraek/urls.py        (routes / to core.urls)
↓
core/urls.py              (routes '' to core.views.home)
↓
core/views.py → home()    (returns HttpResponse)
↓
HTTP Response
```

Currently `home()` returns a plain text string. The real views — run detail, route map, stats — are not yet built.

### Deployment

The app runs on **Vercel** via `vercel.json`:

```json
{
  "builds": [{ "src": "soundtraek/asgi.py", "use": "@vercel/python" }],
  "routes": [
    { "src": "/static/(.*)", "dest": "/static/$1" },
    { "src": "/(.*)", "dest": "soundtraek/asgi.py" }
  ]
}
```

Static files (`/static/`) are served by Whitenoise directly. Everything else goes to the Django ASGI app.

Locally you run: `python manage.py runserver`
In CI: GitHub Actions runs `pytest` on every push (`.github/workflows/ci.yml`)

### Data files (not in the repo)

Historical data lives outside the repo at:
`~/Library/CloudStorage/Dropbox/Projects/`

This includes 447 Spotify CSVs, ~300 Nike Run Club JSON files, and 233 Garmin `.fit` files spanning January 2023 to January 2026. These are imported via management commands into the local SQLite database.

---

## Design rationale

**Why Vercel for a Django app?** Low friction for a solo project — push to GitHub, it deploys. The `@vercel/python` builder handles Django's ASGI interface directly.

**Why SQLite locally?** Zero setup, zero cost, fast iteration. The schema is designed to migrate to Supabase (Postgres) when the app goes to wider beta — UUID PKs and no SQLite-specific features are used.

**Why keep data files in Dropbox, not the repo?** The raw data is hundreds of MBs of personal activity files. It doesn't belong in version control. Management commands import from a local path.

---

## Best practice audit

| Check | Status | Note |
|---|---|---|
| `.env` not committed | ✅ | Listed in `.gitignore` |
| `db.sqlite3` not committed | ✅ | Listed in `.gitignore` |
| `static/` directory exists | ✅ | Created — suppresses Whitenoise warning |
| `templates/` directory referenced in settings | ✅ | `BASE_DIR / 'templates'` |
| `DEBUG=True` default | ⚠️ | Fine locally; must be `False` in production |
| `SECRET_KEY` fallback hardcoded | ⚠️ | Fine locally; must use env var in production |
| Password validators empty | ⚠️ | `AUTH_PASSWORD_VALIDATORS = []` — add before user accounts go live |

---

## Learning takeaway

SoundTræk uses the standard two-layer Django structure: a project config package (`soundtraek/`) and a feature app (`core/`). The database is SQLite locally with a clear migration path to Supabase. The project is nearly empty at the view layer — the real work so far is in the data model and the import pipeline that will feed it.

---

*Did this section make sense?*
*[ Got it ] [ Needs more detail ] [ Still confused ]*
*Record feedback in `meta/comprehension_signals.json`*
