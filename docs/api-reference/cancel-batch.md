---
title: Cancel a Call Batch
sidebar_label: Cancel a Call Batch
sidebar_position: 7
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `POST /v1/calls/batches/:batchId/cancel`

Cancels all **pending** calls in a batch created via [`POST /v1/calls/bulk`](bulk-create-calls.md). Only calls that have not yet been dialed are cancelled; calls already being dialed or already finished are left untouched.

The `batchId` is the `batch_call_id` returned in the bulk response.

---

## Request

```http
POST https://api.vindy.vinter.me/v1/calls/batches/842/cancel
Authorization: Bearer <api-key>
```

No request body.

## Path parameters

| Parameter | Type | Description |
|---|---|---|
| `batchId` | int | The batch's numeric ID — the `batch_call_id` from [`POST /v1/calls/bulk`](bulk-create-calls.md). |

## Response (200 OK)

```json
{ "batch_call_id": 842, "cancelled_pending_count": 37 }
```

| Field | Type | Description |
|---|---|---|
| `batch_call_id` | int | The cancelled batch's id (the `batchId` you passed in the request). |
| `cancelled_pending_count` | int | How many pending calls were cancelled by this request. |

## Errors

| Status | Code | Description |
|---|---|---|
| `400` | `VALIDATION_FAILED` | `batchId` is not a positive integer. |
| `400` | `ERR_BATCH_NOT_FOUND` | Batch not found or not in your company. |
| `400` | `ERR_BATCH_ALREADY_FINAL` | The batch is no longer running (already finished or cancelled). |
| `400` | `ERR_BATCH_NO_PENDING_CALLS` | The batch is still active, but there are no pending calls left to cancel. |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Auth errors. |

:::note Only pending calls are affected
This endpoint stops calls that haven't started yet. Calls already in progress run to completion, and finished calls are unchanged. The returned `cancelled_pending_count` tells you exactly how many were stopped.
:::

## Examples

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl -X POST https://api.vindy.vinter.me/v1/calls/batches/842/cancel \
  -H "Authorization: Bearer $VINDY_API_KEY"
# → { "batch_call_id": 842, "cancelled_pending_count": 37 }
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
async function cancelBatch(batchId) {
  const response = await fetch(
    `https://api.vindy.vinter.me/v1/calls/batches/${batchId}/cancel`,
    {
      method: "POST",
      headers: { Authorization: `Bearer ${process.env.VINDY_API_KEY}` },
    },
  );

  if (!response.ok) {
    const error = await response.json();
    // ERR_BATCH_ALREADY_FINAL / ERR_BATCH_NO_PENDING_CALLS — nothing left to cancel
    throw new Error(`${error.code}: ${error.message}`);
  }

  const { cancelled_pending_count } = await response.json();
  console.log(`Cancelled ${cancelled_pending_count} pending calls`);
  return cancelled_pending_count;
}

await cancelBatch(842);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def cancel_batch(batch_call_id):
    response = requests.post(
        f"https://api.vindy.vinter.me/v1/calls/batches/{batch_call_id}/cancel",
        headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
    )

    if not response.ok:
        error = response.json()
        # ERR_BATCH_ALREADY_FINAL / ERR_BATCH_NO_PENDING_CALLS — nothing left to cancel
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

    count = response.json()["cancelled_pending_count"]
    print(f"Cancelled {count} pending calls")
    return count

cancel_batch(842)
```

</TabItem>
</Tabs>
