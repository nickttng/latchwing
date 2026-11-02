# 0002: Append-only door events with disciplined time

**Status:** Accepted

## Context
Privacy is the first principle. The data should be useful for planning and useless for identifying anyone.

## Decision
- Each record holds exactly four fields: timestamp, door ID, state (`open` or `closed`), and whether time was synced.
- Records are appended, never edited on the device. They are removed only by the ring-buffer rules in 0003.
- A real-time clock keeps time and is synced only from the signed acknowledgment in 0006.
- Device health (battery, reboots) goes in separate rows that say nothing about people.

## Consequences
- The schema is too small to misuse.
- Corrections happen downstream, not by rewriting history.
- Clock drift is visible instead of silent.

## Rejected
- Cameras, audio, or presence sensors: violate the privacy principle.
- Mutable "session" records (open plus duration): easy to corrupt on power loss.
