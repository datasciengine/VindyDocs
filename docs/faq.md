---
title: FAQ
sidebar_label: FAQ
sidebar_position: 8
---

# FAQ

## Can I see another company's data?

No. Each API key is bound to a single company, and every request is automatically scoped to it. Using another company's `call_id` returns 404 — you can't even tell whether it exists. See [Multi-tenancy](concepts/multi-tenancy.md).

## I lost my API key. Can you recover it?

No. The plain key is shown only once at creation. Create a new key and revoke the old one. See [Authentication](authentication.md).

## Why doesn't a call appear in `POST /v1/calls/list`?

The endpoint only returns calls that are fully finalized — ended **and** with the recording transfer settled. If a call just ended, it may take a short while to appear. Calls that are still in progress never appear. See [no half-baked data](api-reference/list-calls/index.md).

## A recording shows in the Vindy panel but the API says `available: false`. Bug?

Expected. The panel may display recordings from temporary sources; the API only serves recordings from durable storage. The API response is the authoritative contract. See [the explanation](api-reference/list-calls/index.md#recording-not-available).

## `call_recording.available` is `false`. Should I retry?

No — that state is **terminal**. Either no recording was produced, or its transfer permanently failed. See [recording retrieval](guides/recording-retrieval.md).

## Is it safe to retry requests?

Yes. All `GET` endpoints are idempotent, and `POST /v1/calls/list` is a **query, not a mutation** — it has no side effects and is safe to retry. Upsert calls on your side (UNIQUE constraint on `call_id`) and retries become harmless.

Write requests are different: `POST /v1/calls/bulk` creates calls, so blindly retrying it can start a second batch and call people twice (a concurrent retry is blocked by `409 BATCH_IN_PROGRESS`). The cancel endpoints are safe to call again.

## How often should I poll?

No more than once per minute. For continuous syncing, use `from_date` with your last sync time — see [incremental sync](guides/incremental-sync.md).

## Why do my date filters fail with 400?

Most likely a missing timezone (`MISSING_TIMEZONE`) or a non-ISO format (`INVALID_DATE_FORMAT`). See [Filtering & Pagination](api-reference/list-calls/filtering-pagination.md) for accepted and rejected formats.

## How do I report an issue?

Include all of the following — it makes debugging dramatically faster:

- Full HTTP method + URL
- Request headers (**mask the Authorization key**: `Bearer 01902f6e...***`)
- Request body
- Response status + body
- The value of the `X-Request-Id` response header
