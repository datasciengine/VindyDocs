---
title: Webhooks
sidebar_label: Webhooks
sidebar_position: 9
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Webhooks (Event Delivery)

Vindy sends events via **HTTP POST** to the webhook URL configured for your account, so you can react in near-real-time instead of continuously polling [`POST /v1/calls/list`](list-calls/index.md). There are **two event types**:

| `event_type` | Fires when | `data` is |
|---|---|---|
| [`call-ended`](#call-ended) | A call **ends** (completed, failed, or cancelled). | The complete call object. |
| [`batch-ended`](#batch-ended) | A **batch** finishes — every call in it has reached a terminal state. | A batch summary with a per-status breakdown. |

Both share the same delivery semantics (retries, at-least-once, header auth) — see [Behavior](#behavior).

---

## Setup

:::info Webhook setup is not self-service yet
Webhook endpoints (your URL and any custom headers) are currently configured **by the Vindy team**. Self-service management is on the roadmap. To enable webhooks, contact the Vindy team and provide:

- **URL** — must be `https://` (plain `http` is not accepted).
- **Custom headers (optional)** — used to authenticate the request on your side. For example `{"x-api-key": "<your-secret>"}` or `{"Authorization": "Bearer <your-token>"}`. Vindy sends these headers verbatim on every delivery.
:::

## The `call-ended` event {#call-ended}

Vindy sends an HTTP `POST` with a JSON body. The body is a **top-level object** (`event_type`, `delivery_id`, `call_id`) that wraps `data` — the **complete call object**, byte-for-byte the same shape returned by [`GET /v1/calls/:callId`](get-call.md) and by each item in [`POST /v1/calls/list`](list-calls/index.md).

```http
POST <your-webhook-url>
Content-Type: application/json
User-Agent: Vindy-Webhooks/1.0
<your custom headers, e.g. x-api-key: ...>
```

```json
{
  "event_type": "call-ended",
  "delivery_id": "0190aa00-1c5a-7000-8000-abc123def456",
  "call_id": 12345,
  "data": {
    "call_id": 12345,
    "call_status": "completed",
    "call_assistant_id": 7,
    "call_squad_id": null,
    "call_phone_number": "+905551112233",
    "call_bound_type": "outbound",
    "call_started_at": "2026-06-08T10:30:00.000Z",
    "call_ended_at": "2026-06-08T10:31:27.000Z",
    "call_created_at": "2026-06-08T10:29:55.000Z",
    "call_duration_seconds": 87,
    "call_end_reason": "customer-ended-call",
    "call_transcript": "AI: Hi, this is Vindy, your AI assistant. I'd like to ask a few quick questions for our customer satisfaction survey — is now a good time?\nUser: Sure, go ahead.\nAI: Thank you. First, may I ask your age?\nUser: Thirty-two.\n",
    "call_structured_data": {
      "9b1c7e2a-4d3f-4a8b-bc12-5e6f7a8b9c01": {
        "name": "Satisfaction Survey",
        "result": {
          "age": 32,
          "overall_satisfaction": 4,
          "support_speed": 5,
          "would_recommend": true
        }
      }
    },
    "call_metadata": { "crm_contact_id": "CNT-90412" },
    "call_recording": {
      "available": true,
      "url": "https://your-bucket.s3.eu-central-1.amazonaws.com/call-records/...wav?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Expires=86400&X-Amz-Signature=...",
      "expires_at": "2026-06-09T10:31:30.000Z"
    }
  }
}
```

`data.call_transcript` is a single string; each turn within it is separated by a newline (`\n`). JSON escapes those newlines, so the value above shows on one line. Rendered with real line breaks, the transcript above reads:

```text
AI: Hi, this is Vindy, your AI assistant. I'd like to ask a few quick questions for our customer satisfaction survey — is now a good time?
User: Sure, go ahead.
AI: Thank you. First, may I ask your age?
User: Thirty-two.
```

### Top-level fields

| Field | Type | Description |
|---|---|---|
| `event_type` | string | `call-ended` for this event. |
| `delivery_id` | string (UUID) | Stable id for this delivery. It stays the same across every retry attempt of the same event, so you can de-duplicate on it as well as on `call_id`. |
| `call_id` | int | The call's stable id — the same identifier used across the API (e.g. [`GET /v1/calls/:callId`](get-call.md)). It's duplicated at the top level so you can de-duplicate and route the event without parsing `data`. |
| `data` | object | The complete call object — all fields below. |

### `data` — the call object

`data` is the same object returned by [`GET /v1/calls/:callId`](get-call.md):

| Field | Type | Description |
|---|---|---|
| `call_id` | int | Stable call ID (same value as the top-level `call_id`). |
| `call_status` | string | `completed` \| `failed` \| `cancelled` — the outcome, exactly as shown in the panel. |
| `call_assistant_id` | int \| null | Assistant that handled the call (`null` for squad calls). |
| `call_squad_id` | UUID \| null | Squad ID if the call was made via a squad (`null` otherwise). |
| `call_phone_number` | string | Phone number called or calling (E.164 format when available). |
| `call_bound_type` | string \| null | `inbound` \| `outbound` \| `null`. |
| `call_started_at` | ISO 8601 (UTC) \| null | When the call actually started — an ISO-8601 timestamp in **UTC**, with a `Z` suffix and millisecond precision (e.g. `2026-05-15T10:30:00.000Z`). Convert to your local timezone for display. `null` if the call never connected. |
| `call_ended_at` | ISO 8601 (UTC) \| null | When the call ended, in the same ISO-8601 UTC format. `null` if the call never connected. |
| `call_created_at` | ISO 8601 (UTC) | When we created the call record in our system, in the same ISO-8601 UTC format. |
| `call_duration_seconds` | int \| null | Call duration in seconds. |
| `call_end_reason` | string \| null | Technical reason the call ended — a wide enum, see [End reasons](list-calls/index.md#end-reasons). Don't fail on unknown values. |
| `call_transcript` | string \| null | Plain-text transcript. Each turn is prefixed with `AI:` (assistant) or `User:` (caller), turns separated by newlines (`\n`). May be empty or `null` for very short or failed calls. |
| `call_structured_data` | object \| null | AI-extracted data, keyed by each structured output's `id` (from [`GET /v1/assistants`](list-assistants.md)); each value holds that output's `name` and extracted `result`. `null` when none was produced — see [Structured data shapes](list-calls/index.md#structured-data-shapes). |
| `call_metadata` | object \| null | The opaque metadata you sent via [`POST /v1/calls/bulk`](bulk-create-calls.md), echoed back verbatim for correlation. `null` if the call wasn't created with metadata. |
| `call_recording` | object | Recording availability + URL — fields below. |

**`data.call_recording`**

| Field | Type | Description |
|---|---|---|
| `available` | bool | Whether a downloadable recording exists for this call. |
| `url` | string \| absent | 24-hour presigned download URL. Present **only** when `available: true`. |
| `expires_at` | ISO string \| absent | When the URL expires (UTC). Present **only** when `available: true`. |

:::warning The recording URL is generated at send time
`data.call_recording.url` is valid for 24 hours **from the moment the webhook was sent**. If you process the event late — or it arrived on a retry — the URL may already have expired; fetch a fresh one via [`GET /v1/calls/:callId`](get-call.md). When `available` is `false`, no durable recording exists for the call — see [what that means](list-calls/index.md#recording-not-available).
:::

## The `batch-ended` event {#batch-ended}

Fires **once** when a batch — created via [`POST /v1/calls/bulk`](bulk-create-calls.md) — reaches `completed`, i.e. **every call in it has reached a terminal state**. This is how you know all the calls in a batch are done; use the `counts` breakdown for the outcome, then fetch the calls via [`POST /v1/calls/batches/:batchId/calls`](get-batch-calls.md).

:::caution Not sent on cancel
A `batch-ended` event is **not** sent when a batch is cancelled. Cancelling a batch already emits an individual `call-ended` for each cancelled call, so the per-call outcomes still reach you through those events.
:::

The top-level object differs from `call-ended`: it carries `batch_call_id` (**not** `call_id`), and `data` is a **batch summary** rather than a call object.

```json
{
  "event_type": "batch-ended",
  "delivery_id": "0a61f9bd-...",
  "batch_call_id": 842,
  "data": {
    "batch_call_id": 842,
    "status": "completed",
    "total_count": 200,
    "counts": {
      "completed": 180,
      "failed": 12,
      "cancelled": 8,
      "pending": 0,
      "processing": 0
    },
    "created_at": "2026-06-09T23:39:20.298Z"
  }
}
```

### Top-level fields

| Field | Type | Description |
|---|---|---|
| `event_type` | string | `batch-ended` for this event. |
| `delivery_id` | string (UUID) | Stable id for this delivery — the same across every retry attempt. De-duplicate on it or on `batch_call_id`. |
| `batch_call_id` | int | The batch's id, duplicated at the top level for convenience. Use it to de-duplicate and to fetch the batch's calls via [`POST /v1/calls/batches/:batchId/calls`](get-batch-calls.md). |
| `data` | object | The batch summary — fields below. |

### `data` — the batch summary

| Field | Type | Description |
|---|---|---|
| `batch_call_id` | int | The batch's id (same value as the top-level `batch_call_id`). |
| `status` | string | The batch's final status — `completed` for this event. |
| `total_count` | int | Total number of calls in the batch. |
| `counts` | object | Per-status breakdown of the batch's calls — sums to `total_count`. |
| `created_at` | ISO string | When the batch was created. |

**`data.counts`**

| Field | Type | Description |
|---|---|---|
| `completed` | int | Calls that finished successfully. |
| `failed` | int | Calls that ended in failure. |
| `cancelled` | int | Calls that were cancelled. |
| `pending` | int | Calls not yet started. `0` for a completed batch. |
| `processing` | int | Calls still in progress. `0` for a completed batch. |

:::note Same delivery semantics as `call-ended`
`batch-ended` is delivered exactly like `call-ended` — respond `2xx` within ~15s, retried with backoff, at-least-once (de-duplicate by `delivery_id` or `batch_call_id`), no ordering guarantee, and the same custom headers/auth. See [Behavior](#behavior).
:::

## Behavior

- **Respond `2xx`** — within ~15 seconds. A non-`2xx` response or a timeout is treated as a failed delivery and Vindy retries.
- **Retries** — failed deliveries are retried with increasing backoff (roughly `30s → 2m → 10m → 1h`), about 5 attempts in total (over ~1 hour), after which the delivery is marked failed.
- **At-least-once delivery** — under adverse network conditions the same event may arrive more than once. **De-duplicate** by `call_id` (or `delivery_id`) for `call-ended`, and by `batch_call_id` (or `delivery_id`) for `batch-ended`.
- **No ordering guarantee** — events may arrive in a different order than the calls ended in.
- **Recording URL freshness** — `data.call_recording.url` is a 24-hour URL generated at send time. If you process the event late, the URL may have expired — fetch a fresh one via [`GET /v1/calls/:callId`](get-call.md).
- **PII** — the payload may contain phone numbers and transcripts, so your endpoint must be `https`.

:::tip Acknowledge fast, process later
Return `2xx` as soon as you've safely stored the event, then do the heavy work (downloading recordings, updating your systems) asynchronously. This keeps you within the ~15s window and avoids unnecessary retries.
:::

## Verifying authenticity

You verify that a request came from Vindy by **checking the custom header you configured** — for example, confirming the expected `x-api-key` value is present. Vindy does not send an HMAC signature in this version; authentication is done entirely through the headers you define during setup.

## Handling a delivery

A robust handler acknowledges quickly, de-duplicates by `call_id`, and re-fetches a fresh recording URL when needed.

<Tabs groupId="lang">
<TabItem value="node" label="Node.js (Express)">

```javascript
import express from "express";

const app = express();
const seen = new Set(); // back this with a DB / unique constraint in production

app.post("/vindy/webhook", express.json(), async (req, res) => {
  // 1. Verify the request came from Vindy (custom header you configured)
  if (req.get("x-api-key") !== process.env.VINDY_WEBHOOK_SECRET) {
    return res.sendStatus(401);
  }

  const { call_id, data } = req.body;

  // 2. De-duplicate (at-least-once delivery)
  if (seen.has(call_id)) return res.sendStatus(200);
  seen.add(call_id);

  // 3. Acknowledge fast, then process asynchronously
  res.sendStatus(200);

  // 4. Recording URL may have expired — fetch a fresh call if you need it
  void processCall(call_id, data);
});

app.listen(3000);
```

</TabItem>
<TabItem value="python" label="Python (Flask)">

```python
import os
from flask import Flask, request, abort

app = Flask(__name__)
seen = set()  # back this with a DB / unique constraint in production

@app.post("/vindy/webhook")
def vindy_webhook():
    # 1. Verify the request came from Vindy (custom header you configured)
    if request.headers.get("x-api-key") != os.environ["VINDY_WEBHOOK_SECRET"]:
        abort(401)

    body = request.get_json()
    call_id = body["call_id"]

    # 2. De-duplicate (at-least-once delivery)
    if call_id in seen:
        return "", 200
    seen.add(call_id)

    # 3. Enqueue for async processing, then acknowledge fast
    enqueue_processing(call_id, body["data"])
    return "", 200
```

</TabItem>
</Tabs>

:::note Related
Webhooks complement, but do not replace, [`POST /v1/calls/list`](list-calls/index.md). For a polling-based reconciliation pattern, see the [Incremental Sync guide](../guides/incremental-sync.md).
:::
