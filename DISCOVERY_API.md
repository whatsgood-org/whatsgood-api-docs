# Whats Good — Events Discovery API

A simple, **read-only** API for discovering upcoming local events across **every
Whats Good city**. Built for trip-planning tools and AI agents: look up the
city someone is visiting and get back what's happening while they're there.

This is a read-only surface — there's nothing to create, approve, or manage.

- **Base URL:** `https://whatsgoodapi.up.railway.app`

---

## Authentication

You'll be issued a **read-only API key** by Whats Good. Send it in the
**`X-API-Key`** header (or, if headers aren't possible, the `?api_key=` query
param) on every request:

```bash
curl "https://whatsgoodapi.up.railway.app/discover/cities" \
  -H "X-API-Key: YOUR_KEY"
```

> Want a key? Contact Whats Good at **dustin@checkwhatsgood.com**.

### Errors

| Status | Meaning |
|--------|---------|
| `401` | Missing or invalid API key |
| `403` | This key isn't a discovery (read-only) key |
| `404` | No such event |
| `422` | Query params failed validation |

---

## What you get

Every endpoint returns only **browsable** events: those that are **upcoming**
(start in the future), not duplicates, and not removed. Events span all cities,
soonest first.

---

## Quickstart

```bash
BASE=https://whatsgoodapi.up.railway.app
KEY="YOUR_KEY"

# 1. See which cities have events right now
curl -s "$BASE/discover/cities" -H "X-API-Key: $KEY"

# 2. Pull upcoming events for a destination
curl -s "$BASE/discover/events?city=St.%20Petersburg" -H "X-API-Key: $KEY"

# 3. Narrow to a trip window and a category
curl -s "$BASE/discover/events?city=Boise&category=Music&start_date_to=2026-07-01" \
  -H "X-API-Key: $KEY"
```

---

## `GET /discover/cities`

List cities that currently have upcoming events, with a count for each. Use it
to discover coverage before querying a destination.

```json
{
  "items": [
    { "city": "Boise", "state": "Idaho", "upcoming_event_count": 641 },
    { "city": "St. Petersburg", "state": "Florida", "upcoming_event_count": 520 }
  ],
  "total": 2
}
```

---

## `GET /discover/events`

List upcoming events across all cities (paginated, soonest first). All params
are optional:

| Param | Purpose |
|-------|---------|
| `city` | City name, e.g. `St. Petersburg` (case-insensitive, exact) |
| `state` | State/region name (case-insensitive, exact) |
| `category` | Category name, e.g. `Music` (case-insensitive, exact) |
| `is_free` | `true` = only free events, `false` = only paid |
| `start_date_to` | Upper bound on start date (ISO 8601). The lower bound is always "now" |
| `search` | Free-text match on title/description |
| `page`, `page_size` | Pagination (`page_size` max **100**, default 20) |

```json
{
  "items": [
    {
      "id": 50274,
      "title": "Movement Mondays",
      "description": "A weekly community dance session.",
      "start_date": "2026-06-01T19:00:00Z",
      "end_date": "2026-06-01T21:00:00Z",
      "city": "St. Petersburg",
      "state": "Florida",
      "venue_name": "The Studio",
      "venue_address": "123 Central Ave",
      "latitude": 27.77,
      "longitude": -82.64,
      "category": "Wellness & Health",
      "is_free": true,
      "price_min": 0,
      "price_max": 0,
      "is_recurring": false,
      "url": "https://example.com/event",
      "images": ["https://…/photo.jpg"]
    }
  ],
  "total": 520,
  "page": 1,
  "page_size": 20,
  "total_pages": 26
}
```

---

## `GET /discover/events/{id}`

Get a single event by its `id` (same shape as one item above). Returns `404` if
the event doesn't exist or isn't browsable.

```bash
curl -s "$BASE/discover/events/50274" -H "X-API-Key: $KEY"
```

---

## Fair use

There are no hard rate limits today, but please keep request volume reasonable —
page through results rather than hammering, and cache where you can. Usage is
tracked per key. Abuse may lead to a key being revoked.

---

Questions or need a key? Contact Whats Good at **dustin@checkwhatsgood.com**.
