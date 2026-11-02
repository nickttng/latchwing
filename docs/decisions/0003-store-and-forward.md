# 0003: Store-and-forward with batched upload

**Status:** Accepted

## Context
Outdoor Wi-Fi is spotty, and radios are the biggest power cost.

## Decision
- Events go first to a ring buffer in on-device flash.
- Upload happens in batches on a schedule, or when the buffer reaches a threshold.
- A batch is cleared only after a signed server acknowledgment (see 0006).
- The transport (Wi-Fi, LoRa, or manual USB pull) stays swappable behind one interface.

## Consequences
- Network outages delay data but don't lose it.
- The radio is on for seconds per day, not continuously.
- If the buffer fills, the oldest records are overwritten. The buffer is sized for at least 30 days offline.

## Rejected
- Sending every event live: drains the battery and loses events when offline.
