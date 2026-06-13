---
title: Cancel a Call
sidebar_label: Cancel a Call
sidebar_position: 6
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `POST /v1/calls/:callId/cancel`

Cancels a single **pending** (not-yet-dialed) call. You can only cancel your own company's calls.

A call can only be cancelled while it is still pending. Calls that are already being dialed, or that have finished, cannot be cancelled.

---

## Request

```http
POST https://api-vindy.vinter.me/v1/calls/12345/cancel
Authorization: Bearer <api-key>
```

No request body.

## Path parameters

| Parameter | Type | Description |
|---|---|---|
| `callId` | int | The call's numeric ID. |

## Response (200 OK)

```json
{ "id": 12345 }
```

| Field | Type | Description |
|---|---|---|
| `id` | int | The ID of the cancelled call. |

## Errors

| Status | Code | Description |
|---|---|---|
| `400` | `VALIDATION_FAILED` | `callId` is not a positive integer. |
| `400` | `ERR_CALL_NOT_FOUND` | Call not found or not in your company. |
| `400` | `ERR_CALL_ALREADY_FINAL` | Call is already final (completed, failed, or cancelled). |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Auth errors. |
| `409` | `ERR_CALL_PROCESSING_CANCEL` | The call is currently being dialed and can no longer be cancelled. |

:::note When cancellation is no longer possible
A call moves quickly from pending to being dialed. If you receive `409 ERR_CALL_PROCESSING_CANCEL`, the call has already started and cannot be stopped via the API. Once it ends you'll see its outcome through [`POST /v1/calls/list`](list-calls/index.md), [`GET /v1/calls/:callId`](get-call.md), or a [webhook event](webhooks.md).
:::

:::tip Cancelling a whole batch
To cancel many pending calls at once — for example every remaining call in a bulk batch — use [`POST /v1/calls/batches/:batchId/cancel`](cancel-batch.md) with the `batch_call_id` from your bulk request, instead of cancelling each call individually.
:::

## Examples

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl -X POST https://api-vindy.vinter.me/v1/calls/12345/cancel \
  -H "Authorization: Bearer $VINDY_API_KEY"
# → { "id": 12345 }
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
async function cancelCall(callId) {
  const response = await fetch(
    `https://api-vindy.vinter.me/v1/calls/${callId}/cancel`,
    {
      method: "POST",
      headers: { Authorization: `Bearer ${process.env.VINDY_API_KEY}` },
    },
  );

  if (!response.ok) {
    const error = await response.json();
    if (error.code === "ERR_CALL_PROCESSING_CANCEL") {
      return false; // too late — the call is already being dialed
    }
    throw new Error(`${error.code}: ${error.message}`);
  }

  const { id } = await response.json();
  return id === callId;
}

await cancelCall(12345);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def cancel_call(call_id):
    response = requests.post(
        f"https://api-vindy.vinter.me/v1/calls/{call_id}/cancel",
        headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
    )

    if not response.ok:
        error = response.json()
        if error.get("code") == "ERR_CALL_PROCESSING_CANCEL":
            return False  # too late — the call is already being dialed
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

    return response.json()["id"] == call_id

cancel_call(12345)
```

</TabItem>
</Tabs>
