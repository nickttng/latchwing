# Review 5, round 01

Narrow check of `addendum-4.md` only, by the user's decision: (1) is each `review-4.md` fix applied faithfully, and (2) does addendum-4 directly contradict addendum-3, addendum-2, addendum, the verdict or the brief's hard constraints. Storage details stay deferred to the firmware storage spec. Date: 2026-11-03.

## Result: needs changes (two wording fixes)

**Hard constraints:** none failed. Nothing adds sensors, live feeds or identification. M3 narrows what is stored. M4 removes the test route after the gate.

## Items

| Item | Status | Note |
|---|---|---|
| K1 | Applied | Matches the fix word for word, with both gates. Replaces addendum-3 R1's `low_batt_skip` example. |
| K2 | Applied, with a contradiction | Option (a) and the gate are in. Addendum-3 R3 still says a sector is "never erased while it holds an entry the unsent data still needs, except under the 90-day rule". Addendum-4 replaces only "keep the current key", so that clause stays and conflicts with erasing the oldest sector (issue 2). |
| K3 | Applied | Oldest-entry rule, the reset increment and both gates. M8 is folded in. |
| Q1 | Applied | Verbatim. |
| Q2 | Applied | Deferred reset row, batch-start stamp, 200-reset gate. |
| Q3 | Applied | "At most" the largest step, the 20 s / W2-RSSI sizing, wrapped keys inside the padding. T5 stands. |
| X1 | Not applied as given | The added first sentence, "Raw data is deleted 90 days after it was written", restates the claim X1 said is broken. It contradicts the same line's server clause (90 days plus the outage) and addendum-3 T2 ("past 90 wall days" for an unpowered unit) (issue 1). |
| X2 | Applied | |
| X3 | Applied | "On this board", plus the one-time public-key print. |
| M1–M7, M9, M10 | Applied | M3 doesn't conflict with the hourly hotspot try (addendum-2) or the `rssi` health field: the device still scans, and `rssi` comes from the last attempt. M8 is in K3. |

## Issues and fixes

1. **X1:** delete the sentence "Raw data is deleted 90 days after it was written." Or replace it with "Raw data is deleted on these schedules:". Keep the rest of the line as written.
2. **K2:** in R3's exception, change "except under the 90-day rule" to "except under the 90-day rule or when the key log is full (K2, `keylog_full`)".

## After both fixes: pass with conditions

- The firmware storage spec meets R2 (with Q1), R3 (with K2, Q2), R4 and R5 (with Q3), and their gates pass before firmware is written or the bench gates run.
- The user accepts the decision changes to 0001–0004 and 0006 and the three skill departures listed in `review-4.md` "What the user must accept", with X1 as corrected above. 0005 is unchanged.

No other file was edited.
