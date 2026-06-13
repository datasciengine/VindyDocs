---
title: Create a Call Batch
sidebar_label: Create a Call Batch
sidebar_position: 3
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `POST /v1/calls/bulk`

Creates bulk outbound calls to the phone numbers you provide, using one assistant **or** one squad (1–200 numbers per request).

Each call may carry an optional `metadata` object: Vindy does not process it and **returns it verbatim** on each call object in [`POST /v1/calls/list`](list-calls/index.md), [`GET /v1/calls/:callId`](get-call.md), and [webhook events](webhooks.md). Use it to tie a call back to a record in your own system (a CRM contact, an order, a support ticket). See [Metadata](#metadata) for details and limits.

The number calls are placed from is **selected automatically** from a configured number on your account — you don't specify it.

:::info One batch at a time
Only **one bulk batch** can be in progress per account at a time. Submitting a new batch while one is still running returns `409 BATCH_IN_PROGRESS`.
:::

---

## Request

```http
POST https://api-vindy.vinter.me/v1/calls/bulk
Authorization: Bearer <api-key>
Content-Type: application/json

{
  "assistant_id": 7,
  "calls": [
    { "phone_number": "+905551112233", "metadata": { "crm_contact_id": "CNT-90412" } },
    { "phone_number": "+905554445566", "metadata": { "crm_contact_id": "CNT-90413" } }
  ]
}
```

To call with a squad, send `squad_id` instead of `assistant_id`.

## Body parameters

| Field | Type | Required | Description |
|---|---|---|---|
| `assistant_id` | int | **one of** | Assistant that places the calls. Not combinable with `squad_id`. |
| `squad_id` | UUID | **one of** | Squad that places the calls. Not combinable with `assistant_id`. |
| `calls` | array | yes | Call targets (1–200). |
| `calls[].phone_number` | string | yes | Destination Turkish number. See [Phone numbers](#phone-numbers) below. |
| `calls[].metadata` | object | no | Optional key-value object (see [Metadata](#metadata) limits). Returned verbatim. |

:::caution
Send **exactly one** of `assistant_id` / `squad_id`. Sending both or neither is a validation error (`VALIDATION_FAILED`).
:::

### Phone numbers {#phone-numbers}

Only **Turkish** numbers are supported. All of the following are accepted and normalized to `+905551112233`:

| You send | Normalized to |
|---|---|
| `+905551112233` | `+905551112233` |
| `905551112233` | `+905551112233` |
| `05551112233` | `+905551112233` |
| `5551112233` | `+905551112233` |

Common separators — spaces, dashes, and parentheses — are tolerated and stripped, so `+90 555 111 22 33` and `0555-111-2233` are accepted too.

Numbers with a non-`+90` country code (`+1`, `+44`, …) are **rejected** — international calling is not supported in this version. Accepted numbers are stored and dialed in normalized `+90…` form; you see that value later as `call_phone_number` on each call in list, get, and webhook responses.

### Metadata {#metadata}

`metadata` is a free-form key-value object that belongs entirely to **you**. Vindy treats it as an opaque payload: we **never read, parse, validate, or act on its contents**, and it has **no effect** on how a call is placed, routed, or processed. We simply store it and return it to you unchanged on every view of that call — in [`POST /v1/calls/list`](list-calls/index.md), in [`GET /v1/calls/:callId`](get-call.md), and in [webhook events](webhooks.md).

Its only job is **correlation on your side**. Attach whatever identifiers your own systems need to tie a call back to your data — a CRM contact ID, an order number, a campaign tag, your internal request ID, and so on. When the result comes back, you read those same keys off `call_metadata` and route the outcome straight into your CRM, database, or workflow — no need to keep a separate phone-number-to-record mapping.

The only rules are structural, so that we can store and echo it reliably:

| Limit | Value |
|---|---|
| Max keys | 50 |
| Max key length | 40 |
| Max value length | 500 |
| Value types | `string`, `number`, `boolean` |
| Nested objects / arrays | Not allowed |

:::caution Use it for your own keys — and keep PII out
Because Vindy never interprets `metadata`, it is the right place for **your** correlation keys (e.g. `crm_contact_id`, `orderId`, `campaign`). It is **not** the place for personal data (names, phone numbers, ID numbers) — keep those in your own systems and reference them by key instead.
:::

## Response (201 Created)

```json
{
  "batch_call_id": 842,
  "accepted": 200
}
```

| Field | Type | Description |
|---|---|---|
| `batch_call_id` | int | Identifier of the created batch. **Keep it** to cancel the batch later via [`POST /v1/calls/batches/:batchId/cancel`](cancel-batch.md). |
| `accepted` | int | Number of calls queued. |

:::info Correlating results
Per-call IDs are **not** returned here (this avoids large payloads on big batches). Outcomes arrive as calls complete via [`POST /v1/calls/list`](list-calls/index.md) and [webhook events](webhooks.md) — both echo the `metadata` you sent on each call object. Store your `metadata` keys and match them to incoming `call_id`s on your side.
:::

Calls are queued and run in the background. Results (transcript, recording, structured data) become available as each call completes.

:::tip Track a batch's progress
To page through this batch's calls as they finish, use [`POST /v1/calls/batches/:batchId/calls`](get-batch-calls.md). To know the moment the **whole** batch is done — with a per-status breakdown — listen for the [`batch-ended` webhook](webhooks.md#batch-ended).
:::

## Errors

| Status | Code | Description |
|---|---|---|
| `400` | `VALIDATION_FAILED` | Both or neither of `assistant_id` / `squad_id` provided, `calls` empty or over 200, etc. |
| `400` | `INVALID_PHONE_NUMBER` | A `calls[i].phone_number` is not a valid Turkish number. The offending index is in `extensions.details.index`. |
| `400` | `INVALID_METADATA` | A call's metadata violates the limits or uses an invalid value type. The offending index is in `extensions.details.index`. |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Auth errors. |
| `404` | `ASSISTANT_NOT_FOUND` | Assistant not found, not in your company, or not callable. |
| `404` | `SQUAD_NOT_FOUND` | Squad not found, not in your company, or not callable. |
| `409` | `NO_OUTBOUND_PHONE_NUMBER` | No outbound-capable number is configured on your account. Contact the Vindy team. |
| `409` | `BATCH_IN_PROGRESS` | A bulk batch is already running. Retry after it finishes. |

:::caution Atomic request
If **any** number or metadata in the request is invalid, **no calls are created** — the whole request is rejected. Fix the offending entry (see `extensions.details.index`) and resubmit.
:::

## Examples

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl -X POST https://api-vindy.vinter.me/v1/calls/bulk \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "assistant_id": 7,
    "calls": [
      { "phone_number": "+905551112233", "metadata": { "crm_contact_id": "CNT-90412" } },
      { "phone_number": "05554445566", "metadata": { "crm_contact_id": "CNT-90413" } }
    ]
  }'
# → { "batch_call_id": 842, "accepted": 2 }
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
async function createBulkCalls(assistantId, targets) {
  const response = await fetch("https://api-vindy.vinter.me/v1/calls/bulk", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.VINDY_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ assistant_id: assistantId, calls: targets }),
  });

  if (!response.ok) {
    const error = await response.json();
    // INVALID_PHONE_NUMBER / INVALID_METADATA carry extensions.details.index
    throw new Error(`${error.code}: ${error.message}`);
  }

  const { batch_call_id, accepted } = await response.json();
  console.log(`Batch ${batch_call_id} queued ${accepted} calls`);
  return batch_call_id; // keep this to cancel the batch later
}

await createBulkCalls(7, [
  { phone_number: "+905551112233", metadata: { crm_contact_id: "CNT-90412" } },
  { phone_number: "05554445566", metadata: { crm_contact_id: "CNT-90413" } },
]);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def create_bulk_calls(assistant_id, targets):
    response = requests.post(
        "https://api-vindy.vinter.me/v1/calls/bulk",
        headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
        json={"assistant_id": assistant_id, "calls": targets},
    )

    if not response.ok:
        error = response.json()
        # INVALID_PHONE_NUMBER / INVALID_METADATA carry extensions.details.index
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

    body = response.json()
    print(f"Batch {body['batch_call_id']} queued {body['accepted']} calls")
    return body["batch_call_id"]  # keep this to cancel the batch later

create_bulk_calls(7, [
    {"phone_number": "+905551112233", "metadata": {"crm_contact_id": "CNT-90412"}},
    {"phone_number": "05554445566", "metadata": {"crm_contact_id": "CNT-90413"}},
])
```

</TabItem>
</Tabs>

### Call with a squad

```bash
curl -X POST https://api-vindy.vinter.me/v1/calls/bulk \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "squad_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "calls": [
      { "phone_number": "+905551112233", "metadata": { "orderId": "ORD-77" } }
    ]
  }'
```
