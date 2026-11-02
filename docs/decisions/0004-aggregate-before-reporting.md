# 0004: Aggregate before reporting

**Status:** Accepted

## Context
Even door events can feel like surveillance if shown minute by minute, such as "someone opened it at 6:02 a.m."

## Decision
- Reports show counts by time bucket (hour of day, day of week, week).
- Raw event rows stay sealed, readable only by the key holders in 0006. They are never published.
- Gaps (dead battery, outage) are shown as gaps, never as zero.

## Consequences
- Reports answer planning questions such as busy times and restock timing.
- Individual visits can't be picked out of what is shared.

## Rejected
- Publishing raw logs or live "door is open now" feeds.
