---
title: Response Format
sidebar_label: Response Format
sidebar_position: 1
---

# Response Format

All Vindy API responses are JSON (`application/json; charset=utf-8`) and follow a small set of predictable shapes. Learn them once and every endpoint feels familiar.

Most list responses are wrapped in a pagination object — see [Filtering & Pagination](../api-reference/list-calls/filtering-pagination.md#paginated). The one exception is [`GET /v1/assistants`](../api-reference/list-assistants.md), which returns `{ data, total }` instead.

---

## Error format {#error-envelope}

All error responses share the same shape:

```json
{
  "statusCode": 401,
  "timestamp": "2026-06-03T12:34:56.789Z",
  "path": "/v1/assistants",
  "requestId": "01902f6e-7c5a-7000-8000-abc123",
  "code": "INVALID_API_KEY",
  "message": "API key is invalid, expired, or has been revoked.",
  "extensions": {
    "code": "INVALID_API_KEY",
    "statusCode": 401,
    "timestamp": "2026-06-03T12:34:56.789Z",
    "path": "/v1/assistants",
    "requestId": "01902f6e-7c5a-7000-8000-abc123"
  }
}
```

| Field | Type | Description |
|---|---|---|
| `statusCode` | int | HTTP status code. |
| `timestamp` | ISO string | UTC time of the error. |
| `path` | string | Request path. |
| `requestId` | string | UUID for this request — include it in bug reports. |
| `code` | string | Machine-readable error code — see the [Error Codes catalog](../errors.md). |
| `message` | string | Human-readable description. |
| `extensions` | object | Always present. Mirrors `code`, `statusCode`, `timestamp`, `path`, and `requestId`, plus extra detail for some errors (see below). |

`extensions` is present on every error. Beyond the mirrored fields above, it carries extra machine-readable detail for some errors — `extensions.recording_status` on recording-url errors (404/409), or `extensions.details` on date-format and bulk-call errors. For unexpected server errors (`500`) it is also where the code lives, as `extensions.code: "HTTP_500"` (that is the one case with no top-level `code`).

One more optional field can appear:

- `validation_errors` — an array of strings listing exactly what failed validation (on some 400 errors). It also appears inside `extensions`.

---

## Request ID header {#request-id}

Every response carries an `X-Request-Id` header (UUID). Include it in bug reports to speed up debugging.

If you send your own `X-Request-Id` header, the API will honor it — useful for correlating requests across your own logs.
