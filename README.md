# Whats Good — Customer API Docs

Public documentation for the Whats Good events API (city/customer integrations).

- **Docs site (start here):** https://whatsgood-org.github.io/whatsgood-api-docs/ — readable guide
- **Interactive API reference:** https://whatsgood-org.github.io/whatsgood-api-docs/reference.html
- **Narrative guide (source):** [API_GUIDE.md](./API_GUIDE.md)
- **Base URL:** `https://whatsgoodapi.up.railway.app`
- **OpenAPI spec (customer-scoped):** [openapi.json](./openapi.json)

This repo documents **only the customer-accessible surface** (what a per-customer
`X-API-Key` can do). Admin/internal endpoints are intentionally excluded.

## Contents

| File | What |
|------|------|
| `index.html` | Landing page — renders `API_GUIDE.md` as a readable guide |
| `reference.html` | Interactive API reference (Scalar) over `openapi.json` |
| `API_GUIDE.md` | Auth + setup→approve→consume workflow + endpoint examples |
| `openapi.json` | Customer-scoped OpenAPI spec (machine-readable / for agents) |

> The spec here is curated for customers. It's generated from the API's full
> internal spec with admin/internal routes removed.
