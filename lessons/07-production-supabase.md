# Lesson 07 — Going to Production

SQLite works great on your laptop. Here's exactly what needs to change before this app runs on Vercel with a real database.

## Why SQLite Can't Go to Production

Vercel's serverless model uses an **ephemeral filesystem**. Every time you deploy a new version, you get a fresh container. Any files written to disk — including `db.sqlite3` — are created during that deployment and destroyed when the container shuts down after the next cold start.

This works great for static files (handled by Whitenoise) and frontend builds. It breaks *catastrophically* for a database. Your `db.sqlite3` file would be created on deploy, some data written to it, then on the next deployment it's gone — along with all your data.

Supabase (a managed Postgres database) solves this. The database lives *outside* the container, persistent, and accessible from every deployment.

## The Database Switch

Today, `settings.py` has a hardcoded SQLite path:

```python
# Before (SQLite only)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

In production, this needs to switch to Postgres. The modern way to do this is with **environment variables** and a helper library:

```python
# After (Supabase in prod, SQLite locally)
import dj_database_url

DATABASES = {
    'default': dj_database_url.config(
        default=f"sqlite:///{BASE_DIR / 'db.sqlite3'}",
        conn_max_age=600,
        ssl_require=not DEBUG,
    )
}
```

This says:
- **If `DATABASE_URL` env var is set** (production): use it as the connection string (Supabase provides one like `postgres://user:pass@db.example.supabase.co:5432/postgres`)
- **If `DATABASE_URL` is not set** (local development): fall back to `sqlite:///db.sqlite3`
- **`conn_max_age=600`**: Postgres connection pooling (hold connections for 10 minutes, then refresh)
- **`ssl_require=not DEBUG`**: Require SSL in production (Vercel enforces this), but allow plaintext locally

### New Dependencies

Add two packages to `requirements.txt`:

```
dj-database-url==1.3.0
psycopg2-binary==2.9.9
```

- **`dj-database-url`** — parses the Postgres connection string from the env var
- **`psycopg2-binary`** — the Python adapter Django needs to talk to Postgres (vs SQLite's built-in driver)

## Environment Variables: The Complete List

Today, `.env.example` documents a few variables. For production, you need to know them all.

Create or update `.env.example` with this complete list:

```bash
# Django core
DJANGO_SECRET_KEY=your-secret-key-here
DEBUG=False
ALLOWED_HOSTS=localhost,127.0.0.1,yourdomain.vercel.app

# Database
DATABASE_URL=postgres://user:password@db.example.supabase.co:5432/postgres

# Frontend
FRONTEND_URL=http://localhost:5173
CORS_ALLOWED_ORIGINS=http://localhost:5173,http://localhost:5174,http://localhost:5175,https://yourdomain.vercel.app

# Spotify OAuth
SPOTIFY_CLIENT_ID=your-spotify-client-id
SPOTIFY_CLIENT_SECRET=your-spotify-client-secret
SPOTIFY_REDIRECT_URI=http://localhost:8000/auth/spotify/callback

# Strava OAuth
STRAVA_CLIENT_ID=your-strava-client-id
STRAVA_CLIENT_SECRET=your-strava-client-secret
STRAVA_REDIRECT_URI=http://localhost:8000/auth/strava/callback

# Geolocation
GEONAMES_USERNAME=your-geonames-username
```

### The Tricky Ones

**`DATABASE_URL`** — Get this from Supabase project settings → Database → Connection string. Choose "URI" mode (not "psql command"). It looks like:
```
postgres://[user]:[password]@db.example.supabase.co:5432/postgres
```

**`ALLOWED_HOSTS`** — Production: your Vercel domain (e.g., `myapp.vercel.app`). Local: `localhost,127.0.0.1`. In `settings.py`, make it configurable:
```python
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', 'localhost,127.0.0.1').split(',')
```

**`CORS_ALLOWED_ORIGINS`** — Currently hardcoded in `settings.py`:
```python
# Bad: hardcoded
CORS_ALLOWED_ORIGINS = ['http://localhost:5173', 'http://localhost:5174', 'http://localhost:5175']
```

Fix it:
```python
# Good: from env var
CORS_ALLOWED_ORIGINS = os.environ.get('CORS_ALLOWED_ORIGINS', '').split(',') if os.environ.get('CORS_ALLOWED_ORIGINS') else ['localhost:5173', 'localhost:5174', 'localhost:5175']
```

Then in `.env`:
```
CORS_ALLOWED_ORIGINS=http://localhost:5173,https://yourdomain.vercel.app
```

**`FRONTEND_URL`** — Used for OAuth redirects after login. Local: `http://localhost:5173`. Production: your Vercel frontend domain.

## Static Files: collectstatic

Whitenoise (the static file server) uses `CompressedManifestStaticFilesStorage`, which requires running `collectstatic` before serving the app. This step:
1. Finds all static files in Django apps
2. Compresses CSS/JS
3. Generates a manifest file for cache-busting

Add this to your `Procfile`:

```
release: python manage.py collectstatic --noinput && python manage.py migrate --noinput
web: gunicorn soundtraek.wsgi:application
```

The `release` phase runs *before* traffic is routed to the new deployment. If `collectstatic` fails, the release aborts and traffic stays on the old version. This is a safety net.

## CI: Adding a Postgres Service

The current `.github/workflows/ci.yml` runs tests against SQLite. Once you switch to `dj-database-url`, you need a Postgres service in CI:

```yaml
name: Run tests
on:
  push:
  pull_request:
permissions:
  contents: read
jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Migrate
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/postgres
          DJANGO_SECRET_KEY: test-secret-key
          DEBUG: 'False'
        run: python manage.py migrate --noinput

      - name: Run tests
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/postgres
          DJANGO_SECRET_KEY: test-secret-key
          DEBUG: 'False'
        run: pytest
```

The `services:` block spins up a Postgres container for the test run. Each step that needs the database sets `DATABASE_URL` to point at it.

## Postgres vs SQLite: What to Watch

Django abstracts most differences, but a few things behave differently:

### String Sorting

SQLite:
```sql
SELECT * FROM track ORDER BY name;
-- Returns: apple, Apple, Banana, banana
```

Postgres:
```sql
SELECT * FROM track ORDER BY name;
-- Returns: Apple, Banana, apple, banana
```

SQLite sorts case-insensitively by default. Postgres sorts by byte value (uppercase comes before lowercase). **Impact:** If your UI relies on a specific sort order, the visual result will differ. Not a data integrity issue, just a presentation detail.

### JSON Fields

The `Album.palette_json` field uses Django's `JSONField`:

```python
class Album(models.Model):
    palette_json = models.JSONField(default=list)
```

- **SQLite:** Stored as text, queried as text
- **Postgres:** Stored as `jsonb` (binary JSON), indexed and queryable

No code changes needed—Django handles both. But on Postgres, you get faster queries and can use JSON operators (`@>`, `?`, etc.) if you want to add features later.

### UUID Fields

`Run` and `PlayEvent` both have:
```python
id = models.UUIDField(primary_key=True, default=uuid.uuid4)
```

- **SQLite:** Stored as `CHAR(32)`
- **Postgres:** Stored as native `uuid` type

Django handles the conversion transparently. No code changes.

### Data Migration: How to Move Existing Data

You cannot copy `db.sqlite3` to Supabase directly. The file format is incompatible.

**Option 1: Dump and Load (keeps all data)**
```bash
# On local, export all data to JSON
python manage.py dumpdata > backup.json

# Switch to Supabase (update DATABASE_URL in .env or settings)
# Run migrations on the empty Postgres schema
python manage.py migrate

# Load the backup
python manage.py loaddata backup.json
```

**Option 2: Re-import from Source Files (cleanest, but re-runs imports)**

Since all the import commands are idempotent (they check for duplicates before inserting), you can just re-run them:

```bash
# After switching to Supabase
python manage.py import_spotify_csvs /path/to/spotify/
python manage.py import_nrc_runs /path/to/nrc/
python manage.py import_fit_files /path/to/fit/
python manage.py extract_album_colors
```

Option 2 is cleaner because it re-validates the data and ensures all computed fields (e.g., `Track.assigned_color_hex`) are fresh.

## Supabase Setup Checklist

1. **Create a Supabase project**
   - Go to [supabase.com](https://supabase.com)
   - Create a new project (region: pick one close to you)
   - Wait for the database to initialize (~2 minutes)

2. **Get the connection string**
   - Project settings → Database → Connection string
   - Choose "URI" mode
   - Copy the URL, replace `[YOUR-PASSWORD]` with your actual password
   - Paste into your `.env` as `DATABASE_URL`

3. **Run migrations**
   ```bash
   python manage.py migrate
   ```
   If this succeeds, your Django models are now in Postgres.

4. **Add `DATABASE_URL` to Vercel**
   - Go to Vercel project settings → Environment Variables
   - Add `DATABASE_URL` with the Supabase connection string
   - Do the same for all other prod env vars (`DJANGO_SECRET_KEY`, `DEBUG=False`, etc.)

5. **Deploy to Vercel**
   ```bash
   git push
   ```
   GitHub Actions runs CI, then Vercel deploys automatically.

6. **Verify**
   - Visit your Vercel domain and click through the app
   - Check the logs in Vercel dashboard for errors
   - Spot-check the Music tab — should show your tracks
   - Run a test import if you added new data

## Common Pitfalls

**"FATAL: connection refused" in logs**
- `DATABASE_URL` is not set or is wrong
- Supabase database hasn't initialized yet (wait 5 minutes)
- Firewall is blocking Vercel's IP — Supabase security settings may need adjustment

**"No such column" or "relation does not exist"**
- `migrate` didn't run on Vercel
- Check your `Procfile` — does it have the `release` phase?

**Case-sensitive sort breaks the UI**
- Postgres and SQLite sort differently
- Fix by adding `.order_by('name', 'pk')` (secondary sort by PK for determinism)
- Or use PostgreSQL's `COLLATE "C"` if you need byte-order sorts

**"SSL required" error**
- Production expects SSL on the Postgres connection
- Ensure `ssl_require=not DEBUG` in your `DATABASES` config
- Vercel provides SSL automatically

## Evidence

- `soundtraek/settings.py` — database config, env var parsing
- `requirements.txt` — Django, dj-database-url, psycopg2-binary
- `.env.example` — documented env vars
- `Procfile` — release + web phases
- `.github/workflows/ci.yml` — Postgres service setup
- `core/models.py` — models (all compatible with Postgres)

## Next Steps

After Postgres is working in production:

1. **Set up monitoring:** Supabase has a dashboard that shows query performance, connection counts, and storage usage.
2. **Add read replicas:** For scale, Supabase supports read-only replicas in other regions.
3. **Enable point-in-time recovery:** Supabase backups are automatic; configure retention in project settings.
4. **Optimize slow queries:** Use Postgres's `EXPLAIN ANALYZE` to find bottlenecks (mostly relevant once you have a lot of data).

For now, you have a database that will persist, scale with you, and let you deploy confidently to Vercel.
