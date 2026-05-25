# Whats Good — API Guide

A REST API to manage your event pipeline: register **sources** (sites/feeds to
scrape), review the **events** that come in, approve the good ones, and read back
your approved feed.

- **Base URL:** `https://whatsgoodapi.up.railway.app`
- **Interactive reference:** [`/reference.html`](./reference.html)

---

## Authentication

Send your key in the **`X-API-Key`** header (or, if headers aren't possible, the
`?api_key=` query param) on every request:

```bash
curl https://whatsgoodapi.up.railway.app/me \
  -H "X-API-Key: YOUR_KEY"
```

Every request operates on **your account** — you never pass an account id, it's
inferred from your key.

### Errors

| Status | Meaning |
|--------|---------|
| `401` | Missing or invalid API key |
| `404` | No such item in your account |
| `422` | Request body or query params failed validation |

---

## Quickstart: the setup → approve → consume loop

```bash
BASE=https://whatsgoodapi.up.railway.app
KEY="YOUR_KEY"

# 1. Confirm your account
curl -s "$BASE/me" -H "X-API-Key: $KEY"

# 2. Register a source — this AUTO-RUNS a scrape in the background
curl -s -X POST "$BASE/sources" -H "X-API-Key: $KEY" -H "Content-Type: application/json" -d '{
  "name": "City Events Calendar",
  "type": "website",
  "config": {"url": "https://example.com/events"}
}'

# 3. Watch the run (use the source id from step 2)
curl -s "$BASE/sources/123/logs?limit=1" -H "X-API-Key: $KEY"

# 4. Review what was scraped (new events arrive as status="new")
curl -s "$BASE/events?status=new&page_size=50" -H "X-API-Key: $KEY"

# 5. Approve the good ones (single, or bulk)
curl -s -X POST "$BASE/events/456/approve" -H "X-API-Key: $KEY"
curl -s -X POST "$BASE/events/approve" -H "X-API-Key: $KEY" -H "Content-Type: application/json" -d '{"ids":[456,457,458]}'

# 6. Read your live, approved, upcoming feed
curl -s "$BASE/events?status=approved&upcoming=true" -H "X-API-Key: $KEY"
```

**Event lifecycle:** every event (scraped or manually added) starts as
`status="new"`. You review, then move it to `approved` or `rejected`. Approving
an event publishes it to your feed.

---

## Identity

### `GET /me`
Returns your account (city, branding, settings).

```json
{
  "name": "Your City Events",
  "city": "Tampa",
  "state": "Florida",
  "brand_name": "Your City Events",
  "primary_color": "#3B82F6",
  "logo_url": null,
  "allow_venue_creation": false
}
```

---

## Sources

A **source** is one site/feed to scrape. Adding or running a source kicks off a
background scrape; results land in your events as `status="new"`.

> ⚠️ **`POST /sources` and `POST /sources/{id}/run` trigger real scraping.**
> Don't create sources in a loop.

### `GET /sources`
List your sources. Optional filters: `type`, `enabled` (bool).

### `POST /sources`
Add a source **and immediately run it**.

| Field | Required | Notes |
|-------|----------|-------|
| `name` | ✅ | Display name |
| `type` | ✅ | One of `website`, `instagram`, `facebook`, `ticketmaster` |
| `config` | ✅ | Scraper config. For `website`: `{"url": "https://…"}` |
| `venue_matching` | | `broad_source` (default) or `venue_specific` |
| `venue_id` | | Only with `venue_specific` |
| `enabled` | | Default `true` |

```bash
curl -X POST "$BASE/sources" -H "X-API-Key: $KEY" -H "Content-Type: application/json" -d '{
  "name": "Downtown Arts Center",
  "type": "website",
  "config": {"url": "https://example.com/calendar"}
}'
```

Response is the created source (`status` starts `queued`/`ready`, then updates as
the run progresses):

```json
{
  "id": 123, "name": "Downtown Arts Center", "type": "website",
  "config": {"url": "https://example.com/calendar"},
  "venue_matching": "broad_source", "enabled": true,
  "status": "queued", "last_run": null, "last_run_records": null, "last_error": null
}
```

### `GET /sources/{id}`
Get one source.

### `PUT /sources/{id}`
Update fields (`name`, `config`, `enabled`, `venue_matching`, …). Send only what
you want to change.

### `DELETE /sources/{id}`
Delete a source. By default its events are kept (their `source_id` is cleared).
Pass `?clear_events=true` to also delete the source's events.

### `POST /sources/{id}/run`
Re-run a source on demand. Query params: `enhance` (default `true`),
`max_events` (int). Returns a background task handle:

```json
{ "task_id": "run_source_123", "status": "started", "message": "…", "log_id": 9001 }
```

Poll it with [`GET /tasks/{task_id}`](#tasks), or read the run log below.

### `GET /sources/{id}/logs`
Run history for a source. Params: `limit` (default 20, max 100), `offset`. Each
log has scrape counts, status, errors, and enhancement metrics.

### `GET /sources/{id}/logs/{log_id}`
A single run log.

---

## Events

### `GET /events`
List your events (paginated, soonest first). All filters are optional:

| Param | Purpose |
|-------|---------|
| `page`, `page_size` | Pagination (page_size max **100**, default 20) |
| `status` | `new` · `approved` · `rejected` |
| `upcoming` | `true` = only events starting now or later |
| `search` | Match title/description |
| `source`, `source_id` | Filter by source |
| `category_id`, `audience_id`, `venue_id` | Filter by classification |
| `is_free` | `true`/`false` |
| `start_date_from`, `start_date_to` | ISO datetime bounds |
| `enhancement_status` | e.g. `enhanced`, `pending` |
| `include_duplicates` | Default `false` (duplicates hidden) |

```json
{
  "items": [
    {
      "id": 62985, "title": "Friday Night Concert", "source": "website_…",
      "source_id": 905, "start_date": "2026-07-01T19:00:00Z",
      "venue_name": "The Plaza", "is_free": false,
      "category_name": "Music & Concerts", "audience_name": "Locals",
      "status": "new", "enhancement_status": "enhanced",
      "url": "https://…", "images": ["https://…"]
    }
  ],
  "total": 128, "page": 1, "page_size": 20, "total_pages": 7
}
```

### `POST /events`
Manually add a single event (enters as `status="new"`). Only `title` is required.

| Field | Notes |
|-------|-------|
| `title` | ✅ required |
| `description`, `url` | |
| `start_date`, `end_date` | ISO datetime, e.g. `2026-08-15T19:00:00` |
| `venue_name`, `venue_address`, `venue_id` | |
| `latitude`, `longitude` | |
| `is_free`, `price_min`, `price_max` | |
| `categories` | list of category name strings |
| `is_recurring`, `recurrence` | |

```bash
curl -X POST "$BASE/events" -H "X-API-Key: $KEY" -H "Content-Type: application/json" -d '{
  "title": "Saturday Farmers Market",
  "start_date": "2026-08-15T09:00:00",
  "venue_name": "Riverfront Park",
  "is_free": true
}'
```

Returns the created event. Approve it later to make it live.

### `GET /events/{id}`
Full detail for one event.

### `PUT /events/{id}`
Edit an event. Settable fields include `title`, `description`, `start_date`,
`end_date`, venue fields, `is_free`, `price_min`/`price_max`, `category_id`,
`audience_id`, `priority_score` (1–10), `images`, and **`status`**.

`status` must be one of `new`, `approved`, `rejected` (anything else → **422**).

```bash
curl -X PUT "$BASE/events/456" -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"status": "approved"}'
```

### `DELETE /events/{id}`
Permanently delete an event.

### `GET /events/stats`
Counts for your account: totals, by status, by source, by category, upcoming,
duplicates, pending enhancement.

```json
{
  "total_events": 1280, "by_status": {"new": 900, "approved": 350, "rejected": 30},
  "by_source": {"website_…": 600}, "by_category": {"Music & Concerts": 210},
  "upcoming_count": 412, "duplicate_count": 18, "pending_enhancement": 5
}
```

---

## Approvals

In-place approval — no separate publishing step. Approved events are served via
`GET /events?status=approved`.

### `POST /events/{id}/approve`
Set one event to `approved`. Returns the event.

### `POST /events/{id}/reject`
Set one event to `rejected`. Returns the event.

### `POST /events/approve` (bulk)
Approve many at once.

```bash
curl -X POST "$BASE/events/approve" -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"ids": [456, 457, 458]}'
```

```json
{ "approved": [456, 457], "not_found": [458] }
```

`not_found` = ids that don't exist in your account (silently skipped).

### `POST /events/{id}/mark-duplicate`
Mark an event as a duplicate of another of your events:
`{"duplicate_of_id": 457}`.

---

## Lookups

Reference lists for filtering/classifying.

### `GET /categories`
Categories present in your events: `[{"id": 1, "name": "Music & Concerts"}, …]`.

### `GET /audiences`
Audience types you can assign: `[{"id": 1, "name": "Everyone"}, …]`.

---

## Tasks

### `GET /tasks/{task_id}`
Check a background task (e.g. the one returned by `POST /sources/{id}/run`).

```json
{ "task_id": "run_source_123", "status": "completed", "message": "…", "result": {…}, "error": null }
```

`status` ∈ `pending` · `started` · `running` · `completed` · `failed`.

---

## Health

### `GET /health`
No auth. Returns `{"status": "healthy", "database": "connected"}`.

---

Need something that isn't covered here? Contact Whats Good.
