---
title: Filtering & Pagination
sidebar_label: Filtering & Pagination
sidebar_position: 3
---

# Filtering & Pagination

Everything about narrowing and paging through [`POST /v1/calls/list`](index.md): the `cursor`, `limit`, `from_date`, and `to_date` parameters.

Calls are returned **oldest first**, ordered by the moment each call became available to you — not by when the call started.

---

## How the parameters combine

| Request | What you get |
|---|---|
| No `from_date`, `to_date`, `cursor`, or `limit` | The **oldest 100** available calls for the assistant (or squad). If more exist, `has_more` is `true` and `next_cursor` is set — send it back to get the next 100. |
| `limit` only (e.g. `500`) | The oldest *N* calls in a single page (max 500). |
| `from_date` only | Calls available **at or after** `from_date`, oldest first. Continue with `cursor`. |
| `to_date` only | Calls available **strictly before** `to_date`, oldest first. Continue with `cursor`. |
| `from_date` + `to_date` | Calls inside the half-open window `[from_date, to_date)`, oldest first. |
| Any of the above **+ `cursor`** | The **next page** of that same query. Keep every other parameter identical across pages — only `cursor` changes. |

---

## `limit`

- Default **100**, maximum **500**, applied per page.
- A value outside the **1–500** range is rejected with `400 VALIDATION_FAILED`.
- `limit` sets the page size only — it does **not** cap how many calls you can retrieve in total. Keep paging with `cursor` to read everything.

## `cursor` {#cursors}

- Omit it on the **first** request.
- Every response returns `pagination.next_cursor`. While `has_more` is `true`, send that value back as `cursor` to fetch the next page.
- Stop when `has_more` is `false` (at that point `next_cursor` is `null`).
- The cursor is **opaque** — don't build or change it. When you page with a cursor, **resend the same `assistant_id`/`squad_id`, `from_date`, `to_date`, and `limit`**; the cursor only marks your position within that exact query.
- Don't persist cursors long-term (e.g. for days) — use them within a single sync session. For ongoing **incremental** sync, don't save a cursor between runs; instead remember the latest point you've already pulled and pass it as `from_date` on the next run. A cursor marks a position *inside one query*, not a durable watermark. See the [incremental sync guide](../../guides/incremental-sync.md).

```bash
# First request (no cursor)
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"limit":100}'

# Response: { "data": [...], "pagination": { "next_cursor": "X", "has_more": true } }

# Next request (use next_cursor)
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"limit":100,"cursor":"X"}'

# Stop when has_more: false
```

Cursor errors:

| Status | Code | Meaning |
|---|---|---|
| `400` | `INVALID_CURSOR` | Cursor could not be decoded. Use a fresh cursor from a previous response. |
| `400` | `MALFORMED_CURSOR` | Cursor payload doesn't have the expected shape. Don't modify the cursor — use it exactly as returned. |

## The pagination object {#paginated}

Every page is wrapped in the same shape:

```json
{
  "data": [ /* calls */ ],
  "pagination": {
    "next_cursor": "eyJ0IjoiMjAyNi0wNS0xNVQxMDozMToyOC4wMDBaIiwiaSI6MTIzNDZ9",
    "has_more": true,
    "limit": 100
  }
}
```

| Field | Type | Description |
|---|---|---|
| `data` | array | Calls in this page. |
| `pagination.next_cursor` | string \| null | Opaque cursor for the next page. `null` on the last page. |
| `pagination.has_more` | boolean | Whether more calls exist after this page. |
| `pagination.limit` | int | Limit applied in this request. |

---

## Dates: `from_date` / `to_date` {#range-semantics}

Both parameters filter on the moment a call became available to you, as a half-open interval **`[from_date, to_date)`**:

- `from_date` is **inclusive** (>=) — "from this point on"
- `to_date` is **exclusive** (`<`) — "up to but not including this point"

Consecutive windows chain without overlap or gaps. Send either one alone, or both together; omit both to scan from the very beginning.

**Sugar for date-only `to_date`**: `to_date=2026-05-23` (just a date) is automatically expanded to `< 2026-05-24T00:00:00Z`, so **the entire day of May 23 is included**.

`from_date >= to_date` is rejected with `DATE_RANGE_INVALID` (400).

### Accepted formats

All datetime values use **ISO 8601 with UTC as the reference**.

| Format | Example | Meaning |
|---|---|---|
| Date-only | `2026-05-23` | Start of the UTC day (`00:00:00`) |
| UTC datetime | `2026-05-23T15:30:00Z` | UTC 15:30 |
| UTC ms precision | `2026-05-23T15:30:00.123Z` | UTC 15:30 plus 123 ms |
| Datetime with offset | `2026-05-23T15:30:00+03:00` | 15:30 Turkey time (= UTC 12:30) |

### Rejected formats

| Format | Error Code | Problem |
|---|---|---|
| `2026-05-23 15:30:00` | `INVALID_DATE_FORMAT` | Space instead of `T` separator |
| `2026-05-23T15:30:00` | `MISSING_TIMEZONE` | No timezone (`Z` or offset required) |
| `05/23/2026` | `INVALID_DATE_FORMAT` | Not ISO 8601 — year/month/day order is ambiguous |
| `23-05-2026` | `INVALID_DATE_FORMAT` | DD-MM-YYYY is not ISO 8601 |
| `2026-13-01` | `INVALID_DATE_FORMAT` | Invalid month (13) |
| `2026-02-30` | `INVALID_DATE_FORMAT` | Invalid day (February 30) |

Each rejection returns a structured 400 with the error code above, plus machine-readable detail (in `extensions.details`) containing the offending field, what you sent, and example formats. See the [Error Codes catalog](../../errors.md).

### Using Turkey time {#turkey-time}

Turkey is **UTC+3** (fixed year-round; no daylight saving). Three equivalent ways:

```
A) Local Turkey time with offset (recommended):
   "2026-05-23T15:30:00+03:00"

B) Pre-convert to UTC (subtract 3 hours from Turkey time):
   "2026-05-23T12:30:00Z"

C) Pre-convert to UTC with explicit offset:
   "2026-05-23T12:30:00+00:00"
```

**All three refer to the same instant.** The server normalizes to UTC regardless.

### Effective ranges

| Input | Effective Range (UTC) |
|---|---|
| `from_date=2026-05-23` | `>= 2026-05-23T00:00:00Z` |
| `to_date=2026-05-23` (sugar) | `< 2026-05-24T00:00:00Z` |
| `from=2026-05-23` + `to=2026-05-23` | All of May 23 (UTC) |
| `from_date=2026-05-23T15:00:00+03:00` | `>= 2026-05-23T12:00:00Z` |
| `to_date=2026-05-23T18:00:00Z` | `< 2026-05-23T18:00:00Z` (strict) |
| `from=2026-05-23T09:00:00+03:00` + `to=2026-05-23T17:00:00+03:00` | Turkey business hours (UTC 06:00 — 14:00) |

---

## Recipes

**A single day** — the date-only `to_date` sugar includes all of May 23:

```json
{ "assistant_id": 7, "from_date": "2026-05-23", "to_date": "2026-05-23" }
```

**A calendar month** — `to_date` is exclusive, so May 1–31 with no overlap into June:

```json
{ "assistant_id": 7, "from_date": "2026-05-01", "to_date": "2026-06-01" }
```

**Turkey business hours:**

```json
{
  "assistant_id": 7,
  "from_date": "2026-05-23T09:00:00+03:00",
  "to_date": "2026-05-23T17:00:00+03:00"
}
```

**Everything since a point in time** — omit `to_date` to mean "until now":

```json
{ "assistant_id": 7, "from_date": "2026-05-23T11:00:00.000Z" }
```

**Chaining windows without gaps** — because ranges are half-open, one window's `to_date` can be the next window's `from_date`; nothing is counted twice, nothing is skipped:

```json
{ "from_date": "2026-05-23T00:00:00Z", "to_date": "2026-05-24T00:00:00Z" }
{ "from_date": "2026-05-24T00:00:00Z", "to_date": "2026-05-25T00:00:00Z" }
```

### Common pitfalls

| You send | What happens |
|---|---|
| `"2026-05-23 15:30:00"` (space) | `400 INVALID_DATE_FORMAT` — use `T` |
| `"2026-05-23T15:30:00"` (no timezone) | `400 MISSING_TIMEZONE` — add `Z` or `+03:00` |
| `from_date` after `to_date` | `400 DATE_RANGE_INVALID` |
| `"23-05-2026"` or `"05/23/2026"` | `400 INVALID_DATE_FORMAT` — ISO 8601 only |
