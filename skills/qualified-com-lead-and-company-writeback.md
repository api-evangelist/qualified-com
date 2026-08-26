---
name: qualified-com-lead-and-company-writeback
description: >-
  Write enriched lead and account data back into Qualified safely — single upserts, bulk jobs, and
  the blast-radius and reversibility rules that decide when a write needs a human.
api: qualified-com-enterprise-api
generated: '2026-08-26'
method: generated
source: https://app.qualified.com/docs/api
operations:
  - listLeadFields
  - listCompanyFields
  - upsertLead
  - upsertCompany
  - createBulkJob
  - getBulkJob
scopes:
  - lead:manage
  - company:manage
  - bulk_job:manage
---

# Write back to Qualified

Every operationId below is verified against `openapi/qualified-com-enterprise-api-openapi.json`.

## Read this before you write anything

Qualified publishes **no idempotency key** and **no dry-run mode**. What it gives you instead is
natural-key upsert, and that only covers two of the five write paths. Know which one you are on:

| Operation | Key | Reversible? |
|---|---|---|
| `upsertLead` | email address | Replay converges. No delete, no before-image — you cannot restore a value you did not keep. |
| `upsertCompany` | domain | **No.** See the blast radius below. |
| `createBulkJob` | per-item | **No batch rollback.** A blind retry resubmits the whole batch. |
| `cancelMeeting` | Salesforce Event ID | Reverses a booking; itself cannot be undone. Repeat call returns `422`. |
| `createGdprDeletionRequest` | email addresses | **Terminal.** Erasure is the point. |

## Step 1 — Discover the field vocabulary first

Call `listLeadFields` (`GET /v2/leads/fields`) and `listCompanyFields` (`GET /v2/companies/fields`)
before composing any write. Both need only a valid token — no scope — so there is no reason to skip
them. Each returns `id`, `label`, `type`, `name` and `options`.

Writing an unknown field or an invalid value is a `422` with `details.failed_field` naming the
offender. Validating against the field list is cheaper than discovering it in a rejection.

## Step 2 — Upsert a lead

`POST /v2/leads` (`upsertLead`), scope `lead:manage`. Keyed on email address: an existing lead is
updated, a new one is created.

A successful write advances the lead's `updatedAt`, which means it will surface on your own
`updated_after` delta on the next pass. If your sync and your write-back run against the same
warehouse, guard against the echo or you will build a loop.

## Step 3 — Upsert a company, carefully

`POST /v2/companies` (`upsertCompany`), scope `company:manage`. Keyed on domain.

**This is the highest-consequence call on the API.** A single write sets account-level field values
that **every lead on that domain inherits**, and advances `updatedAt` on every one of them. There is
no company read endpoint — "Companies cannot be read back" — so you cannot fetch the current value
before overwriting it, and there is no delete or restore.

Treat it as requiring human confirmation. At minimum, keep your own before-image of what you set,
because Qualified will not give you one.

## Step 4 — Batch with a bulk job

`POST /v2/bulk` (`createBulkJob`), scope `bulk_job:manage`, for batched lead and company writes.

The `202` confirms only that the **batch** was accepted. Individual records inside it can still
fail. Poll `getBulkJob` (`GET /v2/bulk/{id}`) and read:

- `totalRecords`, `processedRecords`, `failedRecords`
- the `errors` on each entry of the job's `result` array

Do not treat `202` as success for the records. Reconcile per item, and repair the failed rows
individually with `upsertLead` / `upsertCompany` rather than resubmitting the whole batch — there is
no rollback and a blind retry re-applies every row that already succeeded.

## Step 5 — Handle errors defensively

Error envelopes differ by status **and** by endpoint. The reference itself says to read `error`,
`code` and `message` defensively rather than branching on one key.

- `400` — malformed request; on bulk it is `{"code","message"}`. Fix and retry.
- `401` `invalid_token` — bad token, or the API is not enabled for the team.
- `403` `insufficient_scope` — re-mint the key with `lead:manage`, `company:manage` or
  `bulk_job:manage` as needed. Granting `:manage` also grants the matching `:view`.
- `422` — write rejected. `details.failed_field` names the field when a single one is at fault.
- `429` `rate_limited` — back off for `Retry-After`. No `Retry-After` means you hit the concurrency
  limit of 10; wait for an in-flight request to finish.
- `500` — retry with backoff.

## Step 6 — GDPR deletion is not a write, it is an erasure

`POST /v2/gdpr_deletion_requests` (`createGdprDeletionRequest`), scope `gdpr:manage`. Up to 5,000
email addresses per request; every address must be well-formed or the whole request is rejected —
that all-or-nothing validation is the only safety net on the call.

There is no undo, no grace period and no restore path. Never wire this to an unattended agent.
