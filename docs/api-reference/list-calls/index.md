---
title: List Calls
sidebar_label: List Calls
sidebar_position: 2
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `POST /v1/calls/list`

Returns the calls placed for one of your assistants or squads — each with its transcript, the data your structured outputs extracted, any metadata you attached, and a recording link when one is ready. Results come back a page at a time via an opaque cursor, and you can narrow them to a date range.

:::info No half-baked data
Only calls that are **ready to be shown to you** are returned. A call is ready when:

- The call has ended (status: `completed`, `failed`, `cancelled`), AND
- If a recording exists, the recording transfer is complete (success or permanently failed)

In-progress calls (e.g., recording transfer still running) are **never** included. This makes your sync logic idempotent.
:::

A call becomes available **shortly after it ends** — once its recording has finished transferring to durable storage. This typically takes a few seconds, and occasionally up to a few minutes for longer recordings. Until a call is ready it appears in neither this list nor the `call-ended` webhook, so a call that just ended may not show up on your very next request.

:::tip Pull and push share the same signal
This endpoint is the **pull** counterpart of the [`call-ended` webhook](../webhooks.md): a call surfaces here and fires that webhook at the same moment it becomes ready. Use the webhook for real-time delivery, and this endpoint to fetch on demand or back-fill anything you may have missed.
:::

---

## Request

```http
POST https://api.vindy.vinter.me/v1/calls/list
Authorization: Bearer <api-key>
Content-Type: application/json

{
  "assistant_id": 7,
  "from_date": "2026-05-01",
  "to_date": "2026-06-01",
  "limit": 100,
  "cursor": null
}
```

## Body parameters

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `assistant_id` | int | **one of** | — | Filter by assistant ID. Mutually exclusive with `squad_id`. |
| `squad_id` | UUID | **one of** | — | Filter by squad ID. Mutually exclusive with `assistant_id`. |
| `from_date` | ISO 8601 string | no | — | Calls that became available at or after this point. See [Filtering & Pagination](filtering-pagination.md). |
| `to_date` | ISO 8601 string | no | — | Calls that became available strictly before this point. See [Filtering & Pagination](filtering-pagination.md). |
| `limit` | int | no | `100` | Maximum items in this page. Range: 1–500. |
| `cursor` | string | no | — | Opaque cursor from a previous `next_cursor`. Omit on the first request. |

**Choosing the filter — `assistant_id` or `squad_id`.** Every call belongs to either an assistant or a squad, so you filter by exactly one of these — never both, and never neither. Pass `assistant_id` to see the calls a single assistant handled; pass `squad_id` to see every call routed through a squad, across all of its member assistants. Both IDs come from [`GET /v1/assistants`](../list-assistants.md).

**Validation rules:**

- One of `assistant_id` or `squad_id` is required; both at once → 400.
- `from_date >= to_date` → 400 (`DATE_RANGE_INVALID`).
- See [Filtering & Pagination](filtering-pagination.md) for date behavior and accepted formats.

## Pagination and filtering

Two independent controls shape the result, and they compose cleanly:

- **The date range** (`from_date` / `to_date`) decides *which* calls are in scope. Both are optional: omit them to scan everything, or set one or both to bound a window.
- **The cursor** (`cursor` / `limit`) walks *through* that scope, one page at a time.

You can use either on its own or both together. With no date range and no cursor, you simply page through all of your calls, **oldest first** — the first request returns the oldest `limit` calls (100 by default), and you keep going until there's nothing left. Add a date range and you page through only that window the same way. In every case the rule is the same: send your filters on the first request, then on each following request send back the `next_cursor` you received — **unchanged** — while keeping `assistant_id`/`squad_id`, `from_date`, `to_date`, and `limit` exactly as they were. The cursor encodes your position *within that specific query*, so changing a filter halfway through a walk produces meaningless results. You're done when `has_more` is `false` (at which point `next_cursor` is `null`).

The full parameter reference, accepted date formats, and copy-paste recipes live in **[Filtering & Pagination](filtering-pagination.md)**.

## Response (200 OK)

```json
{
  "data": [
    {
      "call_id": 12345,
      "call_status": "completed",
      "call_assistant_id": 7,
      "call_squad_id": null,
      "call_phone_number": "+905551112233",
      "call_bound_type": "outbound",
      "call_started_at": "2026-05-15T10:30:00.000Z",
      "call_ended_at": "2026-05-15T10:31:27.000Z",
      "call_created_at": "2026-05-15T10:29:55.000Z",
      "call_duration_seconds": 87,
      "call_end_reason": "customer-ended-call",
      "call_transcript": "AI: Hi, this is Vindy, your AI assistant. I'd like to ask a few quick questions for our customer satisfaction survey — is now a good time?\nUser: Sure, go ahead.\nAI: Thank you. First, may I ask your age?\nUser: Thirty-two.\nAI: How would you rate your overall satisfaction with our service, on a scale of 1 to 5?\nUser: I'd say a four.\nAI: And how would you rate how quickly our team resolved your request, from 1 to 5?\nUser: Five.\nAI: Would you recommend us to a friend or colleague?\nUser: Yes, I would.\nAI: Thank you so much — your feedback means a lot to us. Have a great day!\nUser: You're welcome, you too.\n",
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
        "url": "https://...",
        "expires_at": "2026-06-04T12:34:56.789Z"
      }
    },
    {
      "call_id": 12346,
      "call_status": "completed",
      "call_assistant_id": 7,
      "call_squad_id": null,
      "call_phone_number": "+905554445566",
      "call_bound_type": "outbound",
      "call_started_at": "2026-05-15T11:02:10.000Z",
      "call_ended_at": "2026-05-15T11:02:20.000Z",
      "call_created_at": "2026-05-15T11:01:58.000Z",
      "call_duration_seconds": 10,
      "call_end_reason": "customer-ended-call",
      "call_transcript": "AI: Hi, this is Vindy calling about our customer satisfaction survey. Is now a good time?\nUser: Sorry, wrong number.\n",
      "call_structured_data": null,
      "call_metadata": null,
      "call_recording": {
        "available": false
      }
    }
  ],
  "pagination": {
    "next_cursor": "eyJ0IjoiMjAyNi0wNS0xNVQxMDozMToyOC4wMDBaIiwiaSI6MTIzNDZ9",
    "has_more": true,
    "limit": 100
  }
}
```

`call_transcript` is a single string; each turn within it is separated by a newline (`\n`). JSON escapes those newlines, so the value above shows on one line. Rendered with real line breaks, the first call's transcript reads:

```text
AI: Hi, this is Vindy, your AI assistant. I'd like to ask a few quick questions for our customer satisfaction survey — is now a good time?
User: Sure, go ahead.
AI: Thank you. First, may I ask your age?
User: Thirty-two.
AI: How would you rate your overall satisfaction with our service, on a scale of 1 to 5?
User: I'd say a four.
AI: And how would you rate how quickly our team resolved your request, from 1 to 5?
User: Five.
AI: Would you recommend us to a friend or colleague?
User: Yes, I would.
AI: Thank you so much — your feedback means a lot to us. Have a great day!
User: You're welcome, you too.
```

:::note Failed and cancelled calls are included too
The list returns `completed`, `failed`, and `cancelled` calls — not only successful ones. A call that never connected (for example a `cancelled` call) has no conversation or audio, so its time-based fields are `null` and `call_recording.available` is `false`. Your code should tolerate these nulls:

```json
{
  "call_id": 12347,
  "call_status": "cancelled",
  "call_assistant_id": 7,
  "call_squad_id": null,
  "call_phone_number": "+905557778899",
  "call_bound_type": "outbound",
  "call_started_at": null,
  "call_ended_at": null,
  "call_created_at": "2026-05-15T11:05:00.000Z",
  "call_duration_seconds": null,
  "call_end_reason": null,
  "call_transcript": null,
  "call_structured_data": null,
  "call_metadata": { "crm_contact_id": "CNT-90418" },
  "call_recording": { "available": false }
}
```
:::

## Response fields

**Top-level**

| Field | Type | Description |
|---|---|---|
| `data` | array | Calls in this page. |
| `pagination` | object | Standard [pagination object](filtering-pagination.md#paginated). |

**Call object**

| Field | Type | Description |
|---|---|---|
| `call_id` | int | The call's stable, unique identifier in our system. Use it anywhere an endpoint takes a `:callId` — for example [`GET /v1/calls/:callId`](../get-call.md) to fetch this call, [`GET /v1/calls/:callId/recording-url`](../get-recording-url.md) for a fresh recording link, or [`POST /v1/calls/:callId/cancel`](../cancel-call.md) to cancel it — and to correlate the call with its [`call-ended` webhook](../webhooks.md) payload. |
| `call_status` | string | `completed` \| `failed` \| `cancelled` |
| `call_assistant_id` | int \| null | Assistant ID that handled the call (null for squad calls). |
| `call_squad_id` | UUID \| null | Squad ID if the call was made via a squad. |
| `call_phone_number` | string | Phone number called or calling (E.164 format when available). |
| `call_bound_type` | string \| null | `inbound` \| `outbound` \| `null` |
| `call_started_at` | ISO 8601 (UTC) \| null | When the call actually started — an ISO-8601 timestamp in **UTC**, with a `Z` suffix and millisecond precision (e.g. `2026-05-15T10:30:00.000Z`). Convert to your local timezone for display. `null` if the call never connected. |
| `call_ended_at` | ISO 8601 (UTC) \| null | When the call ended, in the same ISO-8601 UTC format. `null` if the call never connected. |
| `call_created_at` | ISO 8601 (UTC) | When we created the call record in our system, in the same ISO-8601 UTC format. |
| `call_duration_seconds` | int \| null | Call duration in seconds. |
| `call_end_reason` | string \| null | Technical reason the call ended — see [End reasons](#end-reasons) below. |
| `call_transcript` | string \| null | Plain-text transcript of the conversation. Each turn is prefixed with `AI:` (assistant) or `User:` (caller), and turns are separated by newlines (`\n`). May be empty or null for very short or failed calls. |
| `call_structured_data` | object \| null | AI-extracted data, as an object keyed by each structured output applied to the call. Each key is the structured output's `id` returned by [`GET /v1/assistants`](../list-assistants.md), and its value contains the output's `name` and extracted `result`. `null` when no structured data was produced. |
| `call_metadata` | object \| null | The metadata you attached when creating the call via [`POST /v1/calls/bulk`](../bulk-create-calls.md), returned to you verbatim for correlation. `null` if the call was created without metadata. See [Metadata](../bulk-create-calls.md#metadata) for the rules. |
| `call_recording` | object | Recording availability + URL (below). |

**`call_recording` object**

| Field | Type | Description |
|---|---|---|
| `available` | bool | Whether a downloadable recording exists for this call. |
| `url` | string \| absent | 24-hour presigned URL. Present only when `available: true`. |
| `expires_at` | ISO string \| absent | When the URL expires (UTC). Present only when `available: true`. |

### `call_recording.available: false` — what it means {#recording-not-available}

This state is **terminal** — retrying does not help. If a call appears in this list, its recording state is finalized (this endpoint does NOT return calls whose recording transfer is still in progress). `available: false` indicates one of:

- No recording was produced for this call (e.g., very short or failed call where no audio was captured)
- The recording transfer to durable storage **permanently failed**

To inspect the specific reason, call [`GET /v1/calls/:callId/recording-url`](../get-recording-url.md); its `recording_status` field tells you the exact state. Contact the Vindy team if you believe the recording should exist.

:::note Discrepancy with the panel
The Vindy admin panel may display recordings from other sources (e.g., a temporary provider URL). For security, the API only serves recordings from durable storage. Seeing a recording in the panel but not via the API is expected; **the API response is the authoritative customer-facing contract**.
:::

### Structured data shapes

`call_structured_data` returns exactly what your structured output extracts, so a `result` can hold scalars, nested objects, and arrays — including arrays of objects. For example, an *Order Summary* output might return:

```json
{
  "call_structured_data": {
    "c7a1e4d2-8b6f-4c3a-9d10-2f5e7a8b9c34": {
      "name": "Order Summary",
      "result": {
        "customer_name": "Jane Doe",
        "callback_requested": false,
        "orders": [
          { "product": "Wireless Keyboard", "quantity": 2, "in_stock": true },
          { "product": "USB-C Cable", "quantity": 5, "in_stock": false }
        ],
        "shipping": {
          "city": "Istanbul",
          "methods": ["standard", "express"]
        }
      }
    }
  }
}
```

The shape of `result` mirrors the schema you defined for that structured output (returned by [`GET /v1/assistants`](../list-assistants.md)), so you can parse it field by field.

## Call end reasons {#end-reasons}

The `call_end_reason` field is a technical code indicating how the call ended. **It is a wide enum**; the most common values:

| Value | Description |
|---|---|
| `customer-ended-call` | The customer (end-user) hung up |
| `assistant-ended-call` | The assistant ended the call (e.g., conversation concluded naturally) |
| `customer-did-not-answer` | Outbound: no answer |
| `phone-call-provider-closed-websocket` | The telephony provider dropped the connection |
| `exceeded-max-duration` | Maximum call duration reached |
| `silence-timed-out` | Long silence timeout |
| `pipeline-error-*` | An error in the call pipeline (provider, model, etc.) |
Other values may appear — the list grows as new providers and adapters are added. If you keep a known-value list, **don't fail on unknown reasons** — log them and continue.

## Errors

| Status | Code | Description |
|---|---|---|
| `400` | `VALIDATION_FAILED` | `assistant_id` and `squad_id` missing or both provided; `limit` out of range; etc. |
| `400` | `DATE_RANGE_INVALID` | `from_date >= to_date` |
| `400` | `INVALID_DATE_FORMAT` | Date doesn't match accepted formats |
| `400` | `MISSING_TIMEZONE` | Datetime missing `Z` or offset |
| `400` | `INVALID_CURSOR` | Cursor cannot be decoded |
| `400` | `MALFORMED_CURSOR` | Cursor payload has unexpected shape |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Auth errors |
| `500` | `HTTP_500` | Server error — read it from `extensions.code`; include `requestId` when reporting |

## Examples

### Walk all pages

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
# First request (no cursor)
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"limit":100}'

# Response: { "data": [100 calls], "pagination": { "next_cursor": "X", "has_more": true } }

# Next request (use next_cursor)
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"limit":100,"cursor":"X"}'

# Stop when has_more: false
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
async function listAllCalls(assistantId) {
  const calls = [];
  let cursor = undefined;

  do {
    const response = await fetch("https://api.vindy.vinter.me/v1/calls/list", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.VINDY_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ assistant_id: assistantId, limit: 100, cursor }),
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(`${error.code}: ${error.message}`);
    }

    const body = await response.json();
    calls.push(...body.data);
    cursor = body.pagination.next_cursor;
  } while (cursor);

  return calls;
}

const calls = await listAllCalls(7);
console.log(`${calls.length} calls`);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def list_all_calls(assistant_id):
    calls = []
    cursor = None

    while True:
        payload = {"assistant_id": assistant_id, "limit": 100}
        if cursor:
            payload["cursor"] = cursor

        response = requests.post(
            "https://api.vindy.vinter.me/v1/calls/list",
            headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
            json=payload,
        )
        if not response.ok:
            error = response.json()
            raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

        body = response.json()
        calls.extend(body["data"])
        cursor = body["pagination"]["next_cursor"]
        if not cursor:
            break

    return calls

calls = list_all_calls(7)
print(f"{len(calls)} calls")
```

</TabItem>
</Tabs>

### Filter by squad

```bash
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"squad_id":"f47ac10b-58cc-4372-a567-0e02b2c3d479","limit":50}'
```

### Date range — single day

```bash
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "assistant_id": 7,
    "from_date": "2026-05-23",
    "to_date": "2026-05-23"
  }'
```

The date-only `to_date` expands to the start of the next UTC day, so **all of May 23 is included** — see [date sugar](filtering-pagination.md#range-semantics).

### Date range — Turkey business hours

```bash
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "assistant_id": 7,
    "from_date": "2026-05-23T09:00:00+03:00",
    "to_date": "2026-05-23T17:00:00+03:00"
  }'
```

For periodic sync patterns, see the [incremental sync guide](../../guides/incremental-sync.md).
