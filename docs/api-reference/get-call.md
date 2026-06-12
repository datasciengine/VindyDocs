---
title: Get a Call
sidebar_label: Get a Call
sidebar_position: 5
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `GET /v1/calls/:callId`

Returns a single call by its stable `call_id`. The response is **identical to a call object** in [`POST /v1/calls/list`](list-calls/index.md) — including transcript, structured data, metadata, and (if ready) a fresh 24-hour recording URL.

Use it whenever you have a `call_id` — from [List Calls](list-calls/index.md), a [webhook](webhooks.md), or your own records — to pull the call on demand. After a webhook you already have the full object; the usual reason to call this afterward is to refresh a recording URL that has expired.

:::info Same visibility rule as List Calls
Only **customer-visible** calls (status `completed`, `failed`, or `cancelled`) are returned. In-progress / half-baked calls return `404` — exactly the same visibility rule as [`POST /v1/calls/list`](list-calls/index.md).
:::

---

## Request

```http
GET https://api.vindy.vinter.me/v1/calls/12345
Authorization: Bearer <api-key>
```

## Path parameters

| Parameter | Type | Description |
|---|---|---|
| `callId` | int | The call's numeric ID (from [`POST /v1/calls/list`](list-calls/index.md) or a [webhook event](webhooks.md)). |

## Response (200 OK)

Same shape as a [`POST /v1/calls/list`](list-calls/index.md) `data[]` item:

```json
{
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
    "url": "https://...?X-Amz-...",
    "expires_at": "2026-06-09T10:31:30.000Z"
  }
}
```

## Response fields

The call object has the **same fields** as a [List Calls](list-calls/index.md#response-fields) item. The one field worth calling out here:

| Field | Type | Description |
|---|---|---|
| `call_metadata` | object \| null | The metadata you sent via [`POST /v1/calls/bulk`](bulk-create-calls.md), echoed back verbatim. `null` if the call wasn't created with metadata. |

For every other field — `call_status`, `call_transcript`, `call_structured_data`, `call_recording`, the `call_end_reason` enum, and what `call_recording.available: false` means — see the full [List Calls field reference](list-calls/index.md#response-fields).

:::tip Fresh recording URL
The `call_recording.url` returned here is generated **at request time** and is valid for 24 hours. If a webhook payload's recording URL has expired by the time you process it, call this endpoint to get a fresh one.
:::

## Errors

| Status | Code | Description |
|---|---|---|
| `400` | `VALIDATION_FAILED` | `callId` is not a positive integer. |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Auth errors. |
| `404` | `RESOURCE_NOT_FOUND` | Call not found, not yet available (still in progress), or belongs to another company. |

:::note Existence is not leaked
A `call_id` that belongs to another company returns the same `404 RESOURCE_NOT_FOUND` as one that does not exist — see [Multi-tenancy](../concepts/multi-tenancy.md).
:::

## Examples

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl -H "Authorization: Bearer $VINDY_API_KEY" \
  https://api.vindy.vinter.me/v1/calls/12345
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
async function getCall(callId) {
  const response = await fetch(
    `https://api.vindy.vinter.me/v1/calls/${callId}`,
    { headers: { Authorization: `Bearer ${process.env.VINDY_API_KEY}` } },
  );

  if (response.status === 404) {
    return null; // not found, not yet ready, or not in your company
  }
  if (!response.ok) {
    const error = await response.json();
    throw new Error(`${error.code}: ${error.message}`);
  }

  return response.json();
}

const call = await getCall(12345);
console.log(call?.call_status, call?.call_metadata);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def get_call(call_id):
    response = requests.get(
        f"https://api.vindy.vinter.me/v1/calls/{call_id}",
        headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
    )

    if response.status_code == 404:
        return None  # not found, not yet ready, or not in your company
    if not response.ok:
        error = response.json()
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")
    return response.json()

call = get_call(12345)
if call:
    print(call["call_status"], call.get("call_metadata"))
```

</TabItem>
</Tabs>
