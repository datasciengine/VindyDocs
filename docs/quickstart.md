---
title: Quickstart
sidebar_label: Quickstart
sidebar_position: 2
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Quickstart

Make your first Vindy API request in about 5 minutes.

:::info Base URL
Production: `https://api.vindy.vinter.me`
:::

---

## 1. Create an API key

1. Sign into the Vindy panel.
2. Go to **Settings → API Keys** and create a key.
3. The plain key is shown **only once** — save it somewhere safe. If you lose it, you must create a new one.

Store it as an environment variable, never in source code:

```bash
export VINDY_API_KEY="01902f6e-7c5a-7000-8000-abc123def456.R3vP9LkX2nM8jY7fW1qZ4tH6cB0sN5aDmGuI3oV"
```

---

## 2. List your assistants

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl https://api.vindy.vinter.me/v1/assistants \
  -H "Authorization: Bearer $VINDY_API_KEY"
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
const response = await fetch("https://api.vindy.vinter.me/v1/assistants", {
  headers: { Authorization: `Bearer ${process.env.VINDY_API_KEY}` },
});
const body = await response.json();
console.log(body.data);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

response = requests.get(
    "https://api.vindy.vinter.me/v1/assistants",
    headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
)
print(response.json()["data"])
```

</TabItem>
</Tabs>

You'll get your assistants and squads in a single list. Note the `assistant_id` — you need it in the next step:

```json
{
  "data": [
    {
      "type": "assistant",
      "assistant_id": 7,
      "assistant_name": "Customer Support",
      "assistant_language": "tr",
      "assistant_created_at": "2026-05-01T10:30:00.000Z",
      "structured_outputs": [ /* ... */ ]
    }
  ],
  "total": 1
}
```

---

## 3. List your calls

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id": 7, "limit": 10}'
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
const response = await fetch("https://api.vindy.vinter.me/v1/calls/list", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.VINDY_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ assistant_id: 7, limit: 10 }),
});
const body = await response.json();
console.log(body.data);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

response = requests.post(
    "https://api.vindy.vinter.me/v1/calls/list",
    headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
    json={"assistant_id": 7, "limit": 10},
)
print(response.json()["data"])
```

</TabItem>
</Tabs>

Each call includes the transcript, AI-extracted structured data, and — when available — a 24-hour recording URL:

```json
{
  "data": [
    {
      "call_id": 12345,
      "call_status": "completed",
      "call_phone_number": "+905551112233",
      "call_started_at": "2026-05-15T10:30:00.000Z",
      "call_duration_seconds": 87,
      "call_transcript": "AI: Hi, this is Vindy, your AI assistant. I'd like to ask a few quick questions for our customer satisfaction survey — is now a good time?\nUser: Sure, go ahead.\nAI: Thank you. First, may I ask your age?\nUser: Thirty-two.\n",
      "call_structured_data": {
        "9b1c7e2a-4d3f-4a8b-bc12-5e6f7a8b9c01": {
          "name": "Satisfaction Survey",
          "result": { "overall_satisfaction": 4, "would_recommend": true }
        }
      },
      "call_recording": {
        "available": true,
        "url": "https://...",
        "expires_at": "2026-06-04T12:34:56.789Z"
      }
    }
  ],
  "pagination": { "next_cursor": null, "has_more": false, "limit": 10 }
}
```

`call_transcript` is a single string; each turn within it is separated by a newline (`\n`). JSON escapes those newlines, so the value above shows on one line. Rendered with real line breaks, the transcript above reads:

```text
AI: Hi, this is Vindy, your AI assistant. I'd like to ask a few quick questions for our customer satisfaction survey — is now a good time?
User: Sure, go ahead.
AI: Thank you. First, may I ask your age?
User: Thirty-two.
```

---

## 4. Download a recording

If `call_recording.available` is `true`, the `url` field is ready to use — issue a plain GET against it (no auth header needed, the signature is in the URL):

```bash
curl -o call-12345.wav "https://...presigned-url..."
```

The URL expires after 24 hours. Don't store it — generate a fresh one when needed with [`GET /v1/calls/:callId/recording-url`](api-reference/get-recording-url.md).

---

## Next steps

- [Authentication](authentication.md) — key format, security rules, 401 errors
- [Filtering & Pagination](api-reference/list-calls/filtering-pagination.md) — cursor, limit, and date filters for calls
- [Response Format](concepts/response-envelopes.md) — the error shape and request IDs
- [Incremental sync guide](guides/incremental-sync.md) — keeping your database up to date
