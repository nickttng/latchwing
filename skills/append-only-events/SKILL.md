---
name: append-only-events
description: The record format for every door event, written append-only, with trustworthy timestamps. Use when storing, uploading, or reading data, or when handling time.
---

# Append-only events

See decision 0002.

## Record
| Field | Example | Notes |
|---|---|---|
| `ts` | `2026-11-01T14:03:22-05:00` | ISO 8601 with offset |
| `door` | `fridge`, `cabinet-1` | Short fixed ID |
| `state` | `open` or `closed` | |
| `synced` | `true` | Whether the clock was synced |

Health rows use the same log with `door: device` and fields such as `battery_mv` and `boot`.

## Rules
- Append only. Never edit on the device. Only the ring-buffer rules in `store-and-forward` remove records.
- No other fields. Anything new needs a new decision record.
- Store as JSON lines: one record per line, sealed per `access-control`.

## Clock
1. Keep time on the chip's real-time clock through deep sleep.
2. Sync during upload, using only the time in the signed acknowledgment (see `access-control`).
3. After a sync, log the measured drift as a health row.

Downstream:
- Unsynced events can be corrected using the drift from the next sync.
- If drift exceeds 5 minutes, flag the batch instead of quietly fixing it.
