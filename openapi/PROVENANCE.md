# openapi/ provenance

`qualified-com-enterprise-api-openapi.json` — **Qualified Enterprise API 2.0**, OpenAPI 3.0.3,
21 paths / 23 operations / 27 component schemas.

- **Fetched:** 2026-08-26
- **Source:** <https://app.qualified.com/docs/api> — HTTP 200, `text/html`, 857KB.
  Qualified serves the reference as a Redocly page and embeds the complete resolved spec in that
  page's `__redoc_state`. The JSON here is that spec, extracted verbatim and re-serialized with
  indentation. Nothing was added, removed, or rewritten.
- **Why it was not fetched from a spec URL:** Qualified publishes no standalone spec endpoint.
  Probed 2026-08-26 and all missed:
  `api.qualified.com/openapi.json|openapi.yaml|swagger.json|v1/openapi.json|api-docs|docs|redoc` → 404;
  `www.qualified.com/openapi.json|swagger.json` → 404 (HTML);
  `app.qualified.com/openapi.json|docs/api/openapi.json|docs/api.json|docs/api/spec.json` → 401
  (the whole app host is session-gated); `app.qualified.com/docs/api/openapi.yaml` → 403.
  The public reference at <https://www.qualified.com/api> iframes `//app.qualified.com/docs/api`,
  which is the only public carrier of the contract.

## Ownership check (STEP 0c)

The spec belongs to Qualified.com, judged on what it says about itself, not on where it was fetched:

| Signal | Value |
|---|---|
| `info.title` | `Qualified Enterprise API` |
| `servers[0].url` | `https://api.qualified.com` (Production) |
| Auth example | `curl https://api.qualified.com/v2/leads -H "Authorization: Bearer YOUR_API_TOKEN"` |
| Product vocabulary | Leads, visitors, sessions, conversations, meetings, Piper-sent emails — the Qualified product surface |

No sibling-brand host, contact, or terms URL appears anywhere in the document.
