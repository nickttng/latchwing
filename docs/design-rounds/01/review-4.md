# Review 4, round 01

Reviewer: design-review. Scope: the pick in `verdict.md` as amended by `addendum.md`, `addendum-2.md` and `addendum-3.md`. Order of authority: addendum-3, then addendum-2, then addendum, then verdict, then `revision-scrappy.md`. I checked it against the README principles, decisions 0001–0006, the brief's hard constraints, the six skills in `skills/`, and every item in `review-3.md`. By the user's decision, R2–R5 are deferred to a firmware storage spec. I don't block on that deferral, but I do check that their requirements and gates are captured faithfully. Date: 2026-11-03.

## Result: needs changes

**Hard constraints: none changed and none failed.** Nothing adds cameras, audio, presence sensing or identification. Nothing publishes raw rows or adds a live door feed. Only holder keys open sealed data. The brief, `docs/decisions/`, the charters and the skills are unedited (checked against `HEAD`).

Addendum-3 resolves R1, R6, T1, T2, T4, T5, T7, W-A and W-1, and it records R2–R5 as binding requirements with gates. Three new defects must be fixed before firmware is written or the bench gates run (K1–K3). Each needs a sentence or two plus a gate. One of them (K2) is inside the deferred R3 requirement, which as captured contradicts R1, so the spec could not meet both.

## Techniques touched

- `access-control`: rotation (R1), `no_key`, key-log fullness, USB inputs (T1), the logger's board and login (W-A).
- `append-only-events`: the new health fields `no_key`, `keylog_full` and `expired`, and the scan rule (T7).
- `store-and-forward`: chunked uploads (R5), the ring layout (R2), the logger's test route (W-A).
- `time-bucketing`: public-page suppression (T6).
- `enclosure-and-power`: the W1 worst case (T5), `low_batt_skip` and key draws.
- `event-driven-wake`: only through `no_key`, which pauses door slots.

## Status of review-3 items

| Item | Status | Note |
|---|---|---|
| R1 | Resolved, with new defect K1 | The rule, the backstop, the uniqueness bound and the 0006 text are right. The `no_key` stop is not (K1), and R3's `keylog_full` rule breaks it (K2). |
| R2 | Deferred, captured | The figures check: 42 × 96 B = 4,032 B, 64 B spare, 512 × 42 = 21,504 slots, about 96 days at 224 slots a day, and the 30-day floor up to about 345 opens a day. There is no gate (Q1). |
| R3 | Deferred, captured with gaps | The reset-row deferral that makes "a reset loop writes no entries" true is missing (Q2). The `keylog_full` rule contradicts R1 (K2). |
| R4 | Deferred, captured | Matches review-3, gate included. |
| R5 | Deferred, captured with gaps | "Fixed-size chunks at the largest size step" changes review-3's "at most". Nothing ties the largest step to the 20 s cap (Q3). |
| R6 | Resolved | Two gates and the bench card are in. Hardening in M1. |
| T1 | Resolved | Wording in X3. |
| T2 | Resolved | One small wording point in M6. |
| T3 | Resolved in form, new defect K3 | Neither the 2 d term nor the 1 d term is a bound. |
| T4 | Resolved | The 0004 gap list still needs the new causes (X2). |
| T5 | Resolved | 4.50 − 2.44 + 4.89 + 1.33 = 8.28 mAh/day. 1,500 / 8.28 / 1.3 = about 139 days. |
| T6 | Resolved for one "<3" | A residual with two or more "<3" cells remains (M2). Not a firmware or bench blocker. |
| T7 | Resolved | Hotspot hardening in M3. |
| T8 | Partly | The CLI rules are in. The server-copy departure from 0006 has no replacement wording (X1). |
| W-A | Resolved | Conditions in M4. |
| W-1 | Resolved | 96 − 16 − 2 − 1 − 4 = 73 B. |
| W-2 to W-6 | Open, non-blocking | Not taken up. W-2 now affects T1 (X3), and W-3 now underpins R1 (M9). |

## Blocking: fix before firmware and bench gates

- **K1. The `no_key` stop contradicts the key-draw order and, as written, drops door events on cold nights.** The S2 and D2 order draws the key at RF start, before any ADC read. `low_batt_skip` is decided from a reading under load, so by then a key already exists. Read literally, R1's example ("under `low_batt_skip`") skips RF start and stops door slots in every sagging window. That loses counts on exactly the winter nights the trial needs, and it doesn't say how a cause can be logged when there's no key to seal it. **Fix:** "`low_batt_skip` skips association and transfer only. The key is still drawn at RF start in every window. `no_key` applies only when RF start fails or resets the chip. During `no_key` no slots of any kind are written. The first health row under the next key records the `no_key` range in powered hours. On resume the pin is read once and at most one change is logged (as in S3)." **Gates:** add to the sag gate "a window that fires `low_batt_skip` still records door events under a fresh key", and add "a forced RF-start failure gives one `no_key` row with the right range and no slots inside it".
- **K2. R3's "when the log is full, keep the current key" contradicts R1 and the new 0006 text.** The kept key outlives its window, which breaks "its key is never reused after that". The backstop can't help, because drawing a new key needs a key-log entry, and the log is full. So one key could cover a full ring wrap and reuse (key, position) pairs. **Fix to the R3 requirement:** "A full key log never extends a key past its window or the backstop. Then either (a) the oldest key-log sector is erased, crypto-erasing its unsent slots, which matches 0003's oldest-first overwrite, or (b) no slots are written until a sector frees. Either way the report shows a gap with the cause `keylog_full`." I recommend (a). **Gate:** add to the R3 gate "with the key log full, no key covers more than one window".
- **K3. The T3 margin's 2 d and 1 d terms are not bounds.**
  - Measuring by the sector's newest entry relies on a sector spanning 2 d. A `no_key` stretch or a cold spell with missing entries makes the span longer, so the oldest entry outlives 90 days.
  - "1 d for heartbeats lost at resets" covers only 24 resets in 90 days. A brownout each window gives 8 resets a day.
  - **Fix:** "A key-log sector is erased when its *oldest* entry is older than 90 d × (1 − r) − 1 d in powered hours. Every reset that isn't a deep-sleep wake adds 1 h to the powered-hour count." Then lost heartbeats make the count too high, which errs toward early erasure, and the margin no longer depends on how long a sector spans.
  - **Gate:** add "a sector whose oldest entry is past the limit is erased even if its newest isn't", and "10 power-on resets raise the count by at least 10 h".

## Deferred items: requirement gaps to close in addendum-3 now

- **Q1. R2 has no gate.** **Fix:** "Wrap the ring twice on the bench. Every `overwritten` range matches the slots lost, and the layout code asserts that no slot or key-log entry crosses a sector."
- **Q2. R3 dropped review-3's mechanism and leaves a one-bit leak.**
  - "A reset loop writes no entries" holds only if the boot row waits. **Fix:** add "the reset-reason row is deferred to the next hourly row".
  - When an entry is written only once its first slot is due, a stamp taken at write time shows whether a door event came before the next heartbeat after a reset. **Fix:** "Each entry is stamped with the heartbeat count at which its batch began, not when the entry is written."
  - **Gate:** add "200 resets with no door activity add no key-log entries".
- **Q3. R5's chunk rule is ambiguous and doesn't fix the failure it was meant to fix.**
  - "Fixed-size chunks at the largest size step" could mean every upload is the largest step. That would change the W1 budget. **Fix:** "A backlog is sent in chunks of at most the largest size step. Normal uploads keep the size steps." If every upload is meant to be the largest step, say so and recompute T5.
  - Chunking helps only if one chunk fits the 20 s cap. **Fix:** "The largest step is set so that one upload completes within the 20 s cap at the weakest RSSI that passes the W2 gate."
  - "Wrapped keys count inside the padded size."

## Fix in text before the decisions are accepted

- **X1. The 0006 90-day line has no complete replacement, and T8 adds a departure.** A record written on day 1 and uploaded on day 80 stays on R2 storage until day 170. That breaks addendum T3's "90 days after they were written, wherever they are" and 0006's "deleted after 90 days". **Fix:** write one full line: "On the device, 90 powered days less the margin. Pulled copies, 90 days from their oldest record. On the server, 90 days after upload, so up to 90 days plus the outage." Optional: put the oldest slot's age in whole days in the signed upload header, so the Worker can age by write time. That reveals only the outage length, which the server already sees from arrival gaps.
- **X2. The 0004 gap list doesn't name `no_key` or `keylog_full`.** **Fix:** add both to the gap list, next to `expired`.
- **X3. T1 says "can't run again without re-provisioning".** With release-mode flash encryption and no OTA, that means a new board (W-2). **Fix:** "can't run again on this board". Also say that USB prints the identity public key once at enrollment, so "USB accepts only" isn't read as forbidding that output.

## Non-blocking

- **M1. R6 hardening.**
  - `usb_serial_jtag_is_connected()` detects USB start-of-frame packets, so it can read false at power-on, before the host enumerates. Check it after a settle delay of about 1 s.
  - Add the gate "an armed unit with cells fitted and USB connected at power-on starts no window".
  - State that the battery divider sits on the cell side of the 1N5817. The no-cells gate catches it if not.
- **M2. T6 residual.** If a set has two or more "<3" cells, their sum can pin them. For example, two "<3" cells that sum to 4 are both 2. **Fix:** "Keep suppressing the next-smallest cell until the suppressed cells include one of 3 or more." Fix before the first public page.
- **M3. T7 and the hotspot.** The configured hotspot SSID is a holder's phone, so a holder's everyday hotspot could show that holder at the fridge. **Fix:**
  - The hotspot SSID is a dedicated rescue name, turned on only for a rescue. A rescue is then a planned visit, which is already logged.
  - `rssi` is read only at upload attempts, never sampled continuously.
- **M4. W-A conditions.**
  - Run a full chip erase (`esptool.py erase_flash`) after the readout and before provisioning. Otherwise the bench flash-dump gate can find the logger's login and log in unencrypted flash.
  - Make the test route a separate Worker with no access to sealed-batch storage, or delete it after the gate. It is an unauthenticated write route, and the repo is public.
  - In the 0003 text, write "attempt times and results".
- **M5. R1 uniqueness wording.** Say "one window (3 h, or 6 h if the drift test moves uploads)". At 6 h, about 7,206 slots is still under the half-ring backstop.
- **M6. T2 wording.** "Only in total" understates the leak, because torn slots split the totals at power cuts. **Fix:** "only as totals between resets".
- **M7. Naming.** In T8, "R2" means Cloudflare R2, which collides with review item R2. "The T3 text" could mean addendum T3 or review-3 T3. Name both.
- **M8. T3's `r`.** The RC rate varies by chip. Take the worst `r` across every board's own bench run, not only the sacrificial board's.
- **M9. W-3 now underpins R1.** RTC fast memory survives software and watchdog resets. Fold W-3's gate into R1: "a panic reset produces a new key, and the old key is wiped at boot."
- **M10. W-6.** "Cellular is ruled out" is a user preference, not a brief constraint. Label it that way.

## New fields, sensors and data flows

- **`no_key`, `keylog_full`, and `expired` as a powered-hour range:** health fields. They are covered by the D1 list in 0002 (T4), and they need the 0004 gap-list text (X2).
- **The W2 logger and its USB readout:** a new way to move and read data. It is covered by the W-A text for 0003, with the conditions in M4.
- **The logger's test route on the Worker:** a new data flow. It is in the W-A text, and its lifetime needs stating (M4).
- **No new sensors and no live feeds.** USB inputs are covered by T1.

## What the user must accept

**Decision changes, by record** (proposed texts across the verdict and all three addenda; none applied yet):
- **0001:** hourly timer wake that never creates a door event; chatter rule (verdict §6).
- **0002:**
  - clock bullet (verdict §6, addendum T1);
  - sequence number and type sealed (N15);
  - health-field list (D1), now with `expired` as a powered-hour range, `keylog_full` and `no_key`;
  - internal temperature sensor (D2);
  - scan rule (T7).
- **0003:**
  - ack advances the pointer and deletes nothing; fixed schedule; transports (verdict §6);
  - install-mode text (addendum B1);
  - silence email (D3);
  - USB dump text (N5) and trigger text (N9);
  - W2 logger text (W-A).
- **0004:**
  - gap list (verdict §6) plus `clock_suspect`, `expired` (N14), and `no_key` and `keylog_full` (X2);
  - public totals (D5 as replaced by N17, plus T6, plus M2 if taken).
- **0005:** unchanged.
- **0006:**
  - RTC memory and RF-on key draws; 7-day removal; two cloud owners and the keychain rule (verdict §6);
  - same-day revocation (S4);
  - email output rule (D4, N16);
  - dump consequences (N13 as replaced by T2);
  - USB text (T1, with X3);
  - "a batch closes at each scheduled window" (R1);
  - the full 90-day line (X1), which includes the server copy outliving 90 days after an outage.

**Skill departures:**
- `append-only-events`: binary slots instead of JSON lines; the drift rule (F1) replacing "flag drift over 5 minutes" (N22, once 0002 is accepted).
- `store-and-forward`: one try per window instead of retry with backoff (W1).
- `access-control`: for the W2 logger only, an unlocked board with a throwaway login in public for 48 h. It holds no keys and no door data.

**Brief edits:** none required. Recording the Wi-Fi site fact after W2 is optional and the user's call.

**Deferred to the firmware storage spec, with captured requirements and gates:** R2 (with Q1), R3 (with K2 and Q2), R4, R5 (with Q3). Firmware isn't written, and the bench gates don't run, until the spec meets them.
