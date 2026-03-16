# SoundTræk — Project Overview

SoundTræk answers one question: **what song was playing at each moment of a run, and what does that look like on a map?**

It takes your historical running files and Spotify listening history, joins them by timestamp, and renders a GPS route where each segment is colored by the album art of whatever track was playing.

---

## Current state (as of 2026-03-14)

| Layer | Status |
|---|---|
| Database schema | ✅ Designed and migrated |
| Data models | ✅ Written (`core/models.py`) |
| Import pipeline | 🔲 Not yet built |
| Color extraction | 🔲 Not yet built |
| Views / frontend | 🔲 Stub only |

The project has a solid foundation. The next milestone is the import pipeline.

---

## Lessons

| # | Title | What you'll learn |
|---|---|---|
| [01](lessons/01-project-organization.md) | Project Organization | File structure, Django layout, deployment |
| [02](lessons/02-data-flow.md) | Data Flow | How raw files become a colored route |
| [03](lessons/03-storage-model.md) | Storage Model | Schema design decisions, table relationships |

## Reference

- [Architecture](architecture.md)
- [Glossary](glossary.md)
- [Schema reference](../docs/schema.md)
- [Audit report](../audit/architecture-review.md)
