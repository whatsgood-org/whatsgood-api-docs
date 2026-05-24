# Whats Good — Customer API Guide

A REST API for a single city/customer to manage their own event pipeline:
register **sources** (sites/feeds to scrape), review the **events** that come in,
approve the good ones, and read back the approved feed.

> This guide covers **only what a per-customer API key can do.** Admin/ops
> endpoints exist but are not accessible with a customer key — see
> [Out of scope](#out-of-scope).

- **Base URL:** `https://whatsgoodapi.up.railway.app`
- **Interactive reference:** `/docs` (Swagger) · `/redoc` · `/openapi.json`

---

## Authentication

Every request must send your key in the **`X-API-Key`** header (or, if headers
aren't possible, the `?api_key=` query param):

```bash
curl https://whatsgoodapi.up.railway.app/me \
  -H "X-API-Key: wg_live_YOUR_KEY"
```

Your key is **scoped to one customer (city)**. You automatically see and modify
only your own data:

- A `customer_id` is never required from you — it's inferred from the key. If you
  pass one that isn't yours, you get **403**.
- Another tenant's record (by id) responds **404**, never their data.

### Auth errors

| Status | Meaning |
|--------|---------|
| `401` | Missing or invalid API key |
| `403` | Valid key, but you tried to act on another customer |
| `404` | Record doesn't exist **or** belongs to another customer |
| `422` | Request body/params failed validation |

---

## Quickstart: the setup → approve → consume loop

```bash
BASE=https://whatsgoodapi.up.railway.app
KEY="wg_live_YOUR_KEY"

# 1. Confirm which city this key is wired to
curl -s "$BASE/me" -H "X-API-Key: $KEY"

# 2. Register a source — this AUTO-RUNS a scrape in the background
curl -s -X POST "$BASE/sources" -H "X-API-Key: $KEY" -H "Content-Type: application/json" -d '{
  "name": "City Events Calendar",
  "type": "website",
  "config": {"url": "https://example.com/events"}
}'

# 3. Watch the run (use the source id from step 2)
curl -s "$BASE/sources/123/logs?limit=1" -H "X-API-Key: $KEY"

# 4. Review what was scraped (newest events arrive as status="new")
curl -s "$BASE/events?status=new&page_size=50" -H "X-API-Key: $KEY"

# 5. Approve the good ones (single, or bulk)
curl -s -X POST "$BASE/events/456/approve" -H "X-API-Key: $KEY"
curl -s -X POST "$BASE/events/approve" -H "X-API-Key: $KEY" -H "Content-Type: application/json" -d '{"ids":[456,457,458]}'

# 6. Read the live, approved, upcoming feed
curl -s "$BASE/events?status=approved&upcoming=true" -H "X-API-Key: $KEY"
```

**Event lifecycle:** every event (scraped or manually created) starts as
`status="new"`. You review, then move it to `approved` or `rejected`. Approval is
**in place** — the event stays in this database and simply flips to `approved`.

---

## Identity

### `GET /me`
Returns the customer (city) this key belongs to.

```json
{
  "customer_id": "3057627e-d6cf-4cc7-a5e0-d3b56919f721",
  "name": "OpenClaw Demo",
  "city": "Tampa",
  "state": "Florida",
  "brand_name": "OpenClaw Demo",
  "primary_color": "#3B82F6",
  "logo_url": null,
  "allow_venue_creation": false
}
```

---

## Sources

A **source** is one site/feed to scrape. Creating or running a source kicks off a
background scrape; results land in your events as `status="new"`.

> ⚠️ **`POST /sources` and `POST /sources/{id}/run` trigger real scraping**
> (which may call paid third-party services). Don't create sources in a loop.

### `GET /sources`
List your sources. Optional filters: `type`, `enabled` (bool).

### `POST /sources`
Create a source **and immediately run it**.

| Field | Required | Notes |
|-------|----------|-------|
| `name` | ✅ | Display name |
| `type` | ✅ | One of `website`, `instagram`, `facebook`, `ticketmaster` |
| `config` | ✅* | Scraper config. For `website`: `{"url": "https://…"}` |
| `venue_matching` | | `broad_source` (default) or `venue_specific` |
| `venue_id` | | Only with `venue_specific` |
| `enabled` | | Default `true` |

```bash
curl -X POST "$BASE/sources" -H "X-API-Key: $KEY" -H "Content-Type: application/json" -d '{
  "name": "Downtown Arts Center",
  "type": "website",
  "config": {"url": "https://example.com/calendar"},
  "venue_matching": "broad_source"
}'
```

Response is the created source (status starts `queued`/`ready`, then updates as the
run progresses):

```json
{
  "id": 123, "customer_id": "…", "name": "Downtown Arts Center",
  "type": "website", "config": {"url": "https://example.com/calendar"},
  "venue_matching": "broad_source", "enabled": true,
  "status": "queued", "last_run": null, "last_run_records": null,
  "last_error": null, "created_at": "…", "updated_at": "…"
}
```

### `GET /sources/{id}`
Fetch one source.

### `PUT /sources/{id}`
Update fields (`name`, `config`, `enabled`, `venue_matching`, …). Send only what
you want to change.

### `DELETE /sources/{id}`
Delete a source. By default its events are kept (their `source_id` is nulled).
Pass `?clear_events=true` to also delete the source's events.

### `POST /sources/{id}/run`
Re-run a source on demand. Query params: `enhance` (default `true`),
`max_events` (int). Returns a background task handle:

```json
{ "task_id": "run_source_123", "status": "started", "message": "…", "log_id": 9001 }
```

Poll the task with [`GET /tasks/{task_id}`](#tasks), or read the run log below.

### `GET /sources/{id}/logs`
Run history for a source. Params: `limit` (default 20, max 100), `offset`.
Each log has scrape counts, status, errors, and enhancement metrics.

### `GET /sources/{id}/logs/{log_id}`
A single run log.

---

## Events

### `GET /events`
List your events (paginated, newest scheduled first). All filters are optional:

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
Manually create a single event (enters as `status="new"`). Only `title` is required.

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
Counts for your tenant: totals, by status, by source, by category, upcoming,
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

`not_found` = ids that don't exist or aren't yours (silently skipped, never approved).

### `POST /events/{id}/mark-duplicate`
Mark an event as a duplicate of another of your events:
`{"duplicate_of_id": 457}`.

---

## Lookups

Reference lists for filtering/classifying.

### `GET /categories`
Categories present in your events: `[{"id": 1, "name": "Music & Concerts"}, …]`.

### `GET /audiences`
Audience types: `[{"id": 1, "name": "Everyone"}, …]`.

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

## Out of scope

A **customer key cannot** reach these (they require an admin key and return
`401`/`403`): venues, AI chat, source discovery, scheduled jobs, social posts,
tenant-config, the ops dashboard, and all `/admin/*` routes (customer & key
management). Don't call them — your key isn't provisioned for them.

You also cannot create your own customer or issue your own API keys — those are
provisioned for you by a Whats Good admin.
