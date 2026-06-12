---
title: Error Codes
sidebar_label: Error Codes
sidebar_position: 6
---

# Error Codes

Every error carries a machine-readable `code` in `extensions.code`; for the errors in the catalog below it is also mirrored at the top level. Branch on this code in your own `switch/case` logic instead of parsing the human-readable `message`, which may change. Unexpected server errors (`500`) are the one exception with no top-level `code` — read `HTTP_500` from `extensions.code`.

All error responses share the same JSON shape — see the [error format](concepts/response-envelopes.md#error-envelope).

---

## Catalog

| Code | Status | Description | What to do |
|---|---|---|---|
| `MISSING_AUTH_HEADER` | 401 | `Authorization` header is missing. | Add the header. |
| `INVALID_AUTH_FORMAT` | 401 | Header is not in `Bearer <api-key>` format. | Fix the format. |
| `INVALID_API_KEY` | 401 | API key is invalid, expired, or revoked. | Generate a new key. |
| `RESOURCE_NOT_FOUND` | 404 | Resource (e.g. call) not found or not in your company. | Verify the ID. |
| `VALIDATION_FAILED` | 400 | Body or path parameter validation failed. | Fix per `validation_errors`. |
| `INVALID_DATE_FORMAT` | 400 | Date is not in ISO 8601 format. | See [Filtering & Pagination](api-reference/list-calls/filtering-pagination.md). |
| `MISSING_TIMEZONE` | 400 | Datetime is missing a timezone (Z or offset). | Add `Z` or `+HH:MM`. |
| `DATE_RANGE_INVALID` | 400 | `from_date >= to_date` or otherwise invalid. | Fix the range. |
| `INVALID_CURSOR` | 400 | Cursor could not be decoded. | Use a fresh cursor from a previous response. |
| `MALFORMED_CURSOR` | 400 | Cursor payload doesn't have the expected shape. | Don't modify the cursor; use it as returned. |
| `RECORDING_NOT_AVAILABLE` | 404 | The call exists but no recording was ever produced for it. **Terminal.** | Don't retry. Carries `recording_status: "not_found"` — see [recording-url](api-reference/get-recording-url.md#recording-status). |
| `RECORDING_NOT_READY` | 409 | A recording exists but is not downloadable yet — either permanently `failed` (terminal) or still `pending`/`processing` (transient). | Check `recording_status` — see [recording-url](api-reference/get-recording-url.md#recording-status). |
| `INVALID_PHONE_NUMBER` | 400 | A `calls[i].phone_number` is not a valid Turkish number. | Use a Turkish number; the offending index is in `extensions.details.index`. See [Bulk Create Calls](api-reference/bulk-create-calls.md#phone-numbers). |
| `INVALID_METADATA` | 400 | A call's metadata exceeds the limits or uses an invalid value type. | Stay within the limits; the offending index is in `extensions.details.index`. See [Bulk Create Calls](api-reference/bulk-create-calls.md#metadata). |
| `ASSISTANT_NOT_FOUND` | 404 | Assistant not found, not in your company, or not callable. | Verify the `assistant_id`. |
| `SQUAD_NOT_FOUND` | 404 | Squad not found, not in your company, or not callable. | Verify the `squad_id`. |
| `NO_OUTBOUND_PHONE_NUMBER` | 409 | No outbound-capable number is configured on your account. | Contact the Vindy team. |
| `BATCH_IN_PROGRESS` | 409 | A bulk batch is already in progress on your account. | Retry after it finishes. |
| `ERR_CALL_NOT_FOUND` | 400 | Call to cancel was not found or is not in your company. | Verify the `callId`. See [Cancel a Call](api-reference/cancel-call.md). |
| `ERR_CALL_PROCESSING_CANCEL` | 409 | The call is currently being dialed and can no longer be cancelled. | Don't retry; wait for the outcome. See [Cancel a Call](api-reference/cancel-call.md). |
| `ERR_CALL_ALREADY_FINAL` | 400 | The call is already final (completed/failed/cancelled). | Nothing to cancel. See [Cancel a Call](api-reference/cancel-call.md). |
| `ERR_BATCH_NOT_FOUND` | 400 | Batch to cancel was not found or is not in your company. | Verify the `batchId`. See [Cancel a Batch](api-reference/cancel-batch.md). |
| `ERR_BATCH_ALREADY_FINAL` | 400 | The batch is no longer running. | Nothing to cancel. See [Cancel a Batch](api-reference/cancel-batch.md). |
| `ERR_BATCH_NO_PENDING_CALLS` | 400 | No pending calls left in the batch to cancel. | Nothing to cancel. See [Cancel a Batch](api-reference/cancel-batch.md). |
| `HTTP_500` | 500 | Unexpected server error. Read it from `extensions.code` (the top-level `code` is omitted). | Report it with the `requestId`. |
