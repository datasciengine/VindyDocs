---
title: Incremental Sync
sidebar_label: Incremental Sync
sidebar_position: 1
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Incremental Sync

Keep your own database up to date with Vindy calls — without re-downloading everything each time.

---

## One-shot vs ongoing

- **One-shot export** (a historical report): walk all pages via [cursor](../api-reference/list-calls/filtering-pagination.md#cursors), finish.
- **Ongoing sync** (live operations): set `from_date` to your last sync time and fetch only newly-ready calls.

Suggested polling cadence: **no more than once per minute**. More frequent polls are usually wasteful — calls become ready in batches, not milliseconds apart.

---

## The pattern

1. Use a **unique constraint on `call_id`** in your DB and upsert each call.
2. Track a **`last_synced_at`** timestamp on your side — set it to **the moment you start each sync run** (your own UTC clock), *not* a field from the call object.
3. On the next sync, query with `from_date=<last_synced_at>`. `from_date` filters on **when a call became available to you**, so anchoring on your previous run's start time picks up everything that became ready since.
4. Upsert each page. Use the cursor within the session and discard it when done (do not persist long-term).

:::info Why a clock time, not `call_created_at`?
The list is ordered and filtered by **when each call became available to you** — a server-side moment that is **not** exposed on the call object. `call_created_at` is when the call was *queued* (earlier, and unrelated to availability), so using it as the watermark can stall the window or re-scan large overlaps. Anchoring `from_date` on the time you **started** your previous run tracks availability correctly; upserting by `call_id` makes the small boundary overlap harmless. If your clock isn't tightly UTC-synced, subtract a minute or two as a safety margin.
:::

Why this is safe:

- `POST /v1/calls/list` never returns half-processed calls — a call that appears is final. See [no half-baked data](../api-reference/list-calls/index.md).
- Cursor pages never repeat calls within a single walk — the next page does **not** re-return the previous page's calls. You upsert on `call_id` only so that re-running a later sync with an overlapping `from_date` (or retrying a failed request) can't create duplicates.
- `from_date` is inclusive and `to_date` is exclusive, so consecutive windows chain without gaps or overlaps. See [range semantics](../api-reference/list-calls/filtering-pagination.md#range-semantics).

---

## Implementation

<Tabs groupId="lang">
<TabItem value="node" label="Node.js">

```javascript
async function syncCalls(assistantId, lastSyncedAt) {
  // Watermark for the NEXT run: capture your own UTC clock *before* fetching, so
  // calls that become available mid-run are picked up next time. Upsert-by-call_id
  // makes the small boundary overlap harmless.
  const runStartedAt = new Date().toISOString();
  let cursor = undefined;

  do {
    const response = await fetch("https://api.vindy.vinter.me/v1/calls/list", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.VINDY_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        assistant_id: assistantId,
        from_date: lastSyncedAt, // availability watermark — your previous run's start time
        limit: 500,
        cursor,
      }),
    });
    if (!response.ok) {
      const error = await response.json();
      throw new Error(`${error.code}: ${error.message}`);
    }

    const body = await response.json();
    for (const call of body.data) {
      await upsertCall(call); // INSERT ... ON CONFLICT (call_id) DO UPDATE
    }
    cursor = body.pagination.next_cursor;
  } while (cursor);

  return runStartedAt; // save as the next run's lastSyncedAt
}
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
from datetime import datetime, timezone

import requests

def sync_calls(assistant_id, last_synced_at):
    # Watermark for the NEXT run: capture your own UTC clock *before* fetching, so
    # calls that become available mid-run are picked up next time. Upsert-by-call_id
    # makes the small boundary overlap harmless.
    run_started_at = datetime.now(timezone.utc).isoformat()
    cursor = None

    while True:
        payload = {
            "assistant_id": assistant_id,
            "from_date": last_synced_at,  # availability watermark — your previous run's start time
            "limit": 500,
        }
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
        for call in body["data"]:
            upsert_call(call)  # INSERT ... ON CONFLICT (call_id) DO UPDATE

        cursor = body["pagination"]["next_cursor"]
        if not cursor:
            break

    return run_started_at  # save as the next run's last_synced_at
```

</TabItem>
</Tabs>

---

## Day-by-day example

```bash
# Day 1: full backfill
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"limit":500}'
# Walk all pages until has_more: false.
# Save the UTC time you STARTED this run as last_synced_at (not a field from the calls).

# Day 2+: incremental
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"from_date":"<last_synced_at>","limit":500}'
```
