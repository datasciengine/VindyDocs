---
title: Get Recording URL
sidebar_label: Get Recording URL
sidebar_position: 4
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `GET /v1/calls/:callId/recording-url`

Generates a 24-hour temporary (presigned) download URL for a specific call's recording. The URL goes directly to storage and requires no further authentication (the signature is embedded in the URL).

---

## Request

```http
GET https://api.vindy.vinter.me/v1/calls/12345/recording-url
Authorization: Bearer <api-key>
```

## Path parameters

| Parameter | Type | Description |
|---|---|---|
| `callId` | int | The call's numeric ID (from [`POST /v1/calls/list`](list-calls/index.md)). |

## Response (200 OK)

```json
{
  "url": "https://...?X-Amz-Algorithm=...&X-Amz-Signature=...",
  "expires_at": "2026-06-04T12:34:56.789Z"
}
```

| Field | Type | Description |
|---|---|---|
| `url` | string | Presigned URL. Issue an HTTP GET directly against it to download the audio. The signature and validity info are embedded. |
| `expires_at` | ISO string | When the URL expires (UTC, 24 hours from generation). |

## Errors

| Status | Code | Description |
|---|---|---|
| `400` | `VALIDATION_FAILED` | `callId` is not a positive integer. |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Auth errors. |
| `404` | `RESOURCE_NOT_FOUND` | Call not found or not in your company. |
| `404` | `RECORDING_NOT_AVAILABLE` | The call exists, but no recording was ever produced for it. **Terminal** — retrying does not help. Carries `recording_status: "not_found"`. |
| `409` | `RECORDING_NOT_READY` | A recording exists but is not downloadable yet. Carries `recording_status` (inside `extensions`): either `failed` (terminal) or `pending`/`processing` (transient). |

**404 example — no recording was produced (terminal):**

```json
{
  "statusCode": 404,
  "timestamp": "2026-06-03T12:34:56.789Z",
  "path": "/v1/calls/12345/recording-url",
  "requestId": "01902f6e-...",
  "code": "RECORDING_NOT_AVAILABLE",
  "message": "No recording was produced for this call. This is permanent — there is nothing to retrieve, and retrying will not help.",
  "extensions": {
    "code": "RECORDING_NOT_AVAILABLE",
    "statusCode": 404,
    "timestamp": "2026-06-03T12:34:56.789Z",
    "path": "/v1/calls/12345/recording-url",
    "requestId": "01902f6e-...",
    "recording_status": "not_found"
  }
}
```

**409 example — recording still in progress (transient):**

```json
{
  "statusCode": 409,
  "timestamp": "2026-06-03T12:34:56.789Z",
  "path": "/v1/calls/12345/recording-url",
  "requestId": "01902f6e-...",
  "code": "RECORDING_NOT_READY",
  "message": "Recording is not ready yet. The recording transfer is still in progress — retry in a few minutes.",
  "extensions": {
    "code": "RECORDING_NOT_READY",
    "statusCode": 409,
    "timestamp": "2026-06-03T12:34:56.789Z",
    "path": "/v1/calls/12345/recording-url",
    "requestId": "01902f6e-...",
    "recording_status": "processing"
  }
}
```

## `recording_status` values {#recording-status}

Every error in this section carries a `recording_status` (inside `extensions`). It maps to the HTTP status and error code as follows:

| `recording_status` | HTTP | Code | Meaning |
|---|---|---|---|
| `pending` | `409` | `RECORDING_NOT_READY` | Transient (rare) — recording is queued; retry in a few minutes. |
| `processing` | `409` | `RECORDING_NOT_READY` | Transient (rare) — transfer in progress; retry in a few seconds to minutes. |
| `completed` | `200` | — | Success — you receive 200 + URL. (See the note below for a rare exception.) |
| `failed` | `409` | `RECORDING_NOT_READY` | **Terminal** — transfer permanently failed. **Retry does not help.** Contact the Vindy team. |
| `not_found` | `404` | `RECORDING_NOT_AVAILABLE` | **Terminal** — no recording was produced for this call. **Retry does not help.** |

:::note Rare exception to `completed` → 200
A recording marked `completed` whose file is not yet retrievable returns a `409 RECORDING_NOT_READY` carrying `recording_status: "completed"`. This is a transient race condition — retry in a few minutes.
:::

:::info Important
For `call_id`s obtained from [`POST /v1/calls/list`](list-calls/index.md), `recording_status` will almost always be either `completed` (you get a 200) or `failed` (409) / `not_found` (404) — both terminal — because a call doesn't appear in the list until its recording reaches a terminal state. `pending`/`processing` are not part of the normal API flow; they are listed defensively for rare race conditions.
:::

## Notes

- **Do NOT cache the URL**: it expires after 24 hours. Storing it in your DB leads to stale URLs. Generate on demand.
- **Multiple downloads**: you can issue multiple GETs against the same URL within 24 hours. If forwarding to different users, **generate a fresh URL per user**.
- **Terminal vs. transient**: for `call_id`s obtained from [`POST /v1/calls/list`](list-calls/index.md), a not-ready response will almost always be terminal — a `404 RECORDING_NOT_AVAILABLE` (`recording_status: "not_found"`) or a `409 RECORDING_NOT_READY` with `recording_status: "failed"`. Retrying neither helps. In rare race conditions you might see a `409` with `processing`/`pending`; in that case retry after a few minutes.
- **Format**: audio files are `.wav` (mono, 8kHz or 16kHz). Some recordings may use a different codec; check the `Content-Type` header.
- **Size**: typically 1–10 MB; can reach 30 MB for long calls.

## Examples

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
# 1. Get URL
curl -H "Authorization: Bearer $VINDY_API_KEY" \
  https://api.vindy.vinter.me/v1/calls/12345/recording-url
# → { "url": "https://...call.wav?X-Amz-...", "expires_at": "..." }

# 2. Download (quote the URL — query string is long)
curl -o call-12345.wav "https://...call.wav?X-Amz-..."
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
import { writeFile } from "node:fs/promises";

async function downloadRecording(callId) {
  // 1. Get a fresh presigned URL
  const response = await fetch(
    `https://api.vindy.vinter.me/v1/calls/${callId}/recording-url`,
    { headers: { Authorization: `Bearer ${process.env.VINDY_API_KEY}` } },
  );

  if (response.status === 404) {
    const error = await response.json();
    if (error.code === "RECORDING_NOT_AVAILABLE") {
      return null; // terminal — no recording was ever produced
    }
    throw new Error(`${error.code}: ${error.message}`); // RESOURCE_NOT_FOUND
  }
  if (response.status === 409) {
    const error = await response.json();
    const status = error.extensions?.recording_status;
    if (status === "failed") {
      return null; // terminal — transfer permanently failed, do not retry
    }
    throw new Error(`Recording transient state: ${status} — retry later`);
  }
  if (!response.ok) {
    const error = await response.json();
    throw new Error(`${error.code}: ${error.message}`);
  }

  // 2. Download the audio
  const { url } = await response.json();
  const audio = await fetch(url);
  await writeFile(`call-${callId}.wav`, Buffer.from(await audio.arrayBuffer()));
  return `call-${callId}.wav`;
}

await downloadRecording(12345);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def download_recording(call_id):
    # 1. Get a fresh presigned URL
    response = requests.get(
        f"https://api.vindy.vinter.me/v1/calls/{call_id}/recording-url",
        headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
    )

    if response.status_code == 404:
        error = response.json()
        if error.get("code") == "RECORDING_NOT_AVAILABLE":
            return None  # terminal — no recording was ever produced
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")  # RESOURCE_NOT_FOUND
    if response.status_code == 409:
        error = response.json()
        status = error.get("extensions", {}).get("recording_status")
        if status == "failed":
            return None  # terminal — transfer permanently failed, do not retry
        raise RuntimeError(f"Recording transient state: {status} — retry later")
    if not response.ok:
        error = response.json()
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

    # 2. Download the audio
    url = response.json()["url"]
    audio = requests.get(url)
    audio.raise_for_status()

    path = f"call-{call_id}.wav"
    with open(path, "wb") as f:
        f.write(audio.content)
    return path

download_recording(12345)
```

</TabItem>
</Tabs>
