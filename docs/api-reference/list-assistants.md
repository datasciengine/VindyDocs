---
title: List Assistants
sidebar_label: List Assistants
sidebar_position: 1
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `GET /v1/assistants`

Returns your company's assistants and squads in a **single unified list**. Each item has a `type` field (`"assistant"` or `"squad"`) indicating its kind. The **structured output schemas** attached to assistants are included on each item. For squads, structured outputs are the union (deduplicated) of the schemas attached to the squad's member assistants.

---

## Request

```http
GET https://api-vindy.vinter.me/v1/assistants
Authorization: Bearer <api-key>
```

No query parameters.

## Response (200 OK)

```json
{
  "data": [
    {
      "type": "assistant",
      "assistant_id": 7,
      "assistant_name": "Customer Support",
      "assistant_language": "tr",
      "assistant_created_at": "2026-05-01T10:30:00.000Z",
      "structured_outputs": [
        {
          "id": "9b1c7e2a-4d3f-4a8b-bc12-5e6f7a8b9c01",
          "name": "Support Ticket",
          "schema": {
            "type": "object",
            "properties": {
              "customer_name": {
                "type": "string"
              },
              "issue_type": {
                "type": "string"
              },
              "resolved": {
                "type": "boolean"
              }
            }
          }
        }
      ]
    },
    {
      "type": "squad",
      "squad_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "squad_name": "Sales Team",
      "squad_created_at": "2026-05-15T14:20:00.000Z",
      "squad_assistants": [
        { "assistant_id": 7, "assistant_name": "Customer Support" },
        { "assistant_id": 8, "assistant_name": "Sales Assistant" }
      ],
      "structured_outputs": [
        {
          "id": "9b1c7e2a-4d3f-4a8b-bc12-5e6f7a8b9c01",
          "name": "Support Ticket",
          "schema": {
            "type": "object",
            "properties": {
              "customer_name": {
                "type": "string"
              }
            }
          }
        }
      ]
    }
  ],
  "total": 2
}
```

## Response fields

**Top-level**

| Field | Type | Description |
|---|---|---|
| `data` | array | Mixed list of assistant and/or squad items. |
| `total` | int | Size of the `data` array. |

**Assistant item (`type: "assistant"`)**

| Field | Type | Description |
|---|---|---|
| `type` | `"assistant"` | Discriminator. |
| `assistant_id` | int | Stable assistant ID. Use this in [`POST /v1/calls/list`](list-calls/index.md) to filter that assistant's calls, and as the `assistant_id` when launching a batch of outbound calls with [`POST /v1/calls/bulk`](bulk-create-calls.md). |
| `assistant_name` | string | Display name. |
| `assistant_language` | string | Language code (e.g. `tr`, `en`). |
| `assistant_created_at` | ISO string | Creation timestamp (UTC). |
| `structured_outputs` | array | Structured output schemas attached to this assistant. |

**Squad item (`type: "squad"`)**

| Field | Type | Description |
|---|---|---|
| `type` | `"squad"` | Discriminator. |
| `squad_id` | UUID | Stable squad ID. Use this in [`POST /v1/calls/list`](list-calls/index.md) as `squad_id` to filter the squad's calls, and as the `squad_id` when launching a batch of outbound calls with [`POST /v1/calls/bulk`](bulk-create-calls.md). |
| `squad_name` | string \| null | Display name. |
| `squad_created_at` | ISO string | Creation timestamp (UTC). |
| `squad_assistants` | array | Member assistants (short metadata: `assistant_id`, `assistant_name`). |
| `structured_outputs` | array | Deduplicated union of member assistants' structured outputs. |

**StructuredOutput object (same shape in both item types)**

| Field | Type | Description |
|---|---|---|
| `id` | string (UUID) | Stable id of the structured output. A call's extracted values come back under this `id` in `call_structured_data` on [`POST /v1/calls/list`](list-calls/index.md), so you can line each one up with its schema. |
| `name` | string | Display name. |
| `schema` | object | The structured output's JSON Schema — it describes the shape of the data the AI extracts for this output. Almost always an object whose `properties` map each field name to its type, for example `{"type":"object","properties":{"customer_name":{"type":"string"},"resolved":{"type":"boolean"}}}`. The values extracted against it come back under this output's `id` in `call_structured_data` on [List Calls](list-calls/index.md). |

## Errors

| Status | Code |
|---|---|
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` |
| `500` | `HTTP_500` |

## Notes

- The `data` array is ordered: assistants first (`created_at` ASC), then squads (`created_at` ASC).
- The same structured output can appear under multiple assistants and squads (the same `id` may be present in multiple items — expected behavior).
- Extracted values come back under each structured output's `id` in `call_structured_data` on [List Calls](list-calls/index.md), so you can line a call's data up with its schema here.
- Squads are **groups of assistants** — calls made through a squad can be filtered with `squad_id` in [`POST /v1/calls/list`](list-calls/index.md).

## Examples

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl https://api-vindy.vinter.me/v1/assistants \
  -H "Authorization: Bearer $VINDY_API_KEY"
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
const response = await fetch("https://api-vindy.vinter.me/v1/assistants", {
  headers: { Authorization: `Bearer ${process.env.VINDY_API_KEY}` },
});

if (!response.ok) {
  const error = await response.json();
  throw new Error(`${error.code}: ${error.message}`);
}

const { data, total } = await response.json();
console.log(`${total} items`);

for (const item of data) {
  if (item.type === "assistant") {
    console.log(`Assistant #${item.assistant_id}: ${item.assistant_name}`);
  } else {
    console.log(`Squad ${item.squad_id}: ${item.squad_name}`);
  }
}
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

response = requests.get(
    "https://api-vindy.vinter.me/v1/assistants",
    headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
)
if not response.ok:
    error = response.json()
    raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

body = response.json()
print(f"{body['total']} items")

for item in body["data"]:
    if item["type"] == "assistant":
        print(f"Assistant #{item['assistant_id']}: {item['assistant_name']}")
    else:
        print(f"Squad {item['squad_id']}: {item['squad_name']}")
```

</TabItem>
</Tabs>
