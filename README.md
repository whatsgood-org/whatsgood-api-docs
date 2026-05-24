# Whats Good — Customer API Docs

Public documentation for the Whats Good events API (city/customer integrations).

- **API reference (interactive):** https://whatsgood-org.github.io/whatsgood-api-docs/
- **Narrative guide:** [API_GUIDE.md](./API_GUIDE.md)
- **Base URL:** `https://whatsgoodapi.up.railway.app`
- **OpenAPI spec (customer-scoped):** [openapi.json](./openapi.json)

This repo documents **only the customer-accessible surface** (what a per-customer
`X-API-Key` can do). Admin/internal endpoints are intentionally excluded.

## Contents

| File | What |
|------|------|
| `index.html` | Renders `openapi.json` as an interactive reference (Scalar) |
| `openapi.json` | Customer-scoped OpenAPI spec |
| `API_GUIDE.md` | Auth + setup→approve→consume workflow + endpoint examples |

> The spec here is curated for customers. It's generated from the API's full
> internal spec with admin/internal routes removed.
