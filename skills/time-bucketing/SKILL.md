---
name: time-bucketing
description: Summarize events as counts per time window so reports are useful without exposing individual visits. Use when analyzing or publishing data.
---

# Time bucketing

See decision 0004.

## Technique
1. Pair each `open` with the next `closed` to count one visit. Drop unpaired events and note how many.
2. Count visits per bucket: hour of day, day of week, week.
3. Mark buckets with no data (outage, dead battery) as gaps, not zero.
4. Pair each finding with an action, such as moving a stocking day.

## Rules
- Share buckets, never raw rows.
- Use buckets no smaller than one hour in anything published.
