---
name: store-and-forward
description: Buffer events on the device and upload in acknowledged batches. Use when designing storage, upload, or the transport.
---

# Store-and-forward

See decision 0003.

## Technique
1. Append sealed events to a ring buffer in flash (see `access-control`).
2. Upload when the schedule fires (start at every 6 hours) or the buffer is 50 percent full.
3. Send a batch, wait for a signed acknowledgment, then advance the read pointer.
4. No acknowledgment: keep the batch and retry on the next window with backoff.

## Sizing
Buffer size = events per day x bytes per event x 30 days x 2 (margin).

## Transport
Keep one interface (`send(batch) -> ack`). Wi-Fi, LoRa, or a manual USB pull all fit behind it.

## Check
Cut the network for 48 hours and confirm no events are lost.
