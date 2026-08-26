---
name: qualified-com-warehouse-sync
description: >-
  Run a reliable incremental (delta) sync of Qualified engagement data — leads, sessions,
  conversations, messages, meetings and emails — into a warehouse or CDP, respecting Qualified's
  per-resource time filters, availability holds and cursor pagination.
api: qualified-com-enterprise-api
generated: '2026-08-26'
method: generated
source: https://app.qualified.com/docs/api
operations:
  - listLeads
  - listSessions
  - listConversations
  - listConversationMessages
  - listMessages
  - listMeetings
  - listEmails
  - listLeadFields
scopes:
  - lead:view
  - session:view
  - conversation:view
  - meeting:view
  - email:view
---

# Sync Qualified into a warehouse

Every operationId below is verified against `openapi/qualified-com-enterprise-api-openapi.json`.

## Before you start

- Base URL is `https://api.qualified.com`. Authenticate every request with
  `Authorization: Bearer YOUR_API_TOKEN`.
- The key must carry the `:view` scope for each resource you pull. A missing scope is a `403`
  with `{"code": "insufficient_scope"}`, not a `401`.
- A `401` with `{"code": "invalid_token"}` means either the token is wrong **or** the Enterprise
  API is not enabled for the team. Check both before assuming a credential problem.
- Call `listLeadFields` (`GET /v2/leads/fields`) once at the start to learn the tenant's custom
  field vocabulary. It needs no scope, so it is the cheapest way to confirm the token works.

## Step 1 — Backfill, walking backwards

Load history before you switch to deltas. Work in bounded windows — a week, or a day for
high-volume resources — and page each window to completion before moving to the next.

Use the right filter pair per resource; they are not interchangeable:

| Resource | operationId | Filter pair |
|---|---|---|
| Leads | `listLeads` | `created_after` / `created_before` |
| Sessions | `listSessions` | `ended_after` / `ended_before` |
| Conversations | `listConversations` | `ended_after` / `ended_before` |
| Messages | `listMessages` | `created_after` / `created_before` |
| Meetings | `listMeetings` | `created_after` / `created_before` |
| Emails | `listEmails` | `created_after` / `created_before` |

Sessions and conversations have **no** `created_*` filter — both only become available once the
session ends, so they are windowed by end time only.

## Step 2 — Page every window to completion

Pagination is cursor-based and the page size is fixed at 1000. Read `pageInfo` from each response
and pass `endCursor` back as `after` until `hasNextPage` is `false`.

- Drive the loop off `hasNextPage`, never off the row count — a page can return fewer than 1000
  rows and still have a next page.
- On the activity endpoints (`listSessions`, `listConversations`, `listMessages`, `listMeetings`),
  `hasPreviousPage` only reports that the current page is non-empty. Ignore it.
- A cursor whose record is no longer readable is rejected with `400`. Restart that window; do not
  retry the cursor.
- Widening the window is not a substitute for paging. A wide window never returns everything in
  one response.

## Step 3 — Switch to a daily delta

Once you reach the present, record the latest timestamp you saw and pull a rolling ~24-hour window:

- `listLeads`, `listMeetings`, `listEmails` — `updated_after`
- `listConversations`, `listSessions` — `ended_after`
- `listMessages` — `created_after`

Upsert on the record `id`. Re-run the previous window on the next pass to pick up rows that landed
late.

## Step 4 — Respect the availability holds

Records are not readable the moment they exist. A bound that falls inside the hold is rejected with
`400`, not returned empty — so this is a hard failure, not a quiet gap:

- Sessions, and the conversations and messages inside them — available **30 minutes** after the
  session ends.
- Leads and emails — **30 minutes** after creation.
- Meetings — **24 hours** after creation, on every filter. Shift the whole meetings window back a
  day.

Reads by id (`getLead`, `getSession`, `getConversation`, `getMessage`, `getMeeting`, `getEmail`) are
always current and are not held back.

## Step 5 — Resolve identity on every leads pull

This is the step integrations get wrong. Sessions, conversations and meetings carry a `visitorId`
(a browser). Emails carry a `leadId` (a person). The only bridge is `Lead.visitorIds`.

On each `listLeads` pull, map the lead's `visitorIds` onto your stored activity so every visitor's
history points at the right lead. Identification is **retroactive**: because `visitorId` never
changes, a visitor's earlier activity always belonged to that person, and you attach it the moment
the visitor appears in `visitorIds`.

Do not discard activity from unidentified visitors. Each session carries a `visitor` object with
that visitor's field answers and CRM record ids. Note it holds **current** values resolved at
request time, not a snapshot as of the session — re-reading an old session returns today's values.

## Step 6 — Pull transcripts

List conversations on an `ended_after` window, then call `listConversationMessages`
(`GET /v2/conversations/{id}/messages`) per conversation. `senderType` is one of `user`, `visitor`,
`experience`, `ai_profile`; `senderName` is the display name where one exists, and both are null
when the sender cannot be resolved.

Unlike `listMessages`, the per-conversation endpoint is **not** held back, so it returns messages
from a session that is still running.

## Case and time rules

Query parameters are `snake_case` (`updated_after`); response fields are `camelCase` (`updatedAt`).
Timestamps are UTC unless they carry an offset. A bare date like `2026-06-01` means midnight UTC at
the **start** of that day on both bounds — to cover a whole day, set the upper bound to the next
day's date.

## Rate limits

Per team: 10 concurrent, 2,000 per 15 minutes, 7,000 per hour, 120,000 per day. `RateLimit-*`
headers describe the **15-minute tier only**, so you can have remaining quota there and still be
rejected hourly or daily.

On `429`, back off for `Retry-After` seconds. A `429` with **no** `Retry-After` is the concurrency
limit — retry once an in-flight request completes. Keep concurrency at or below 10.
