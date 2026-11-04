# Addendum 3, round 01

This answers `review-3.md`. Order of authority: addendum-3, then addendum-2, then addendum, then verdict, then revision. Date: 2026-11-03.

**Scope, by the user's decision:** this addendum fixes R1, R6, T1–T8, W-A and W-1. R2–R5 are firmware storage details. They are handed to a firmware storage spec, and the review's fixes and gates for them are binding requirements. They are listed below. Firmware isn't written, and the bench gates don't run, until that spec meets them.

Hard constraints: unchanged. Transport: unchanged, and cellular is ruled out.

## Blocking fixes

### R1. Key rotation follows the schedule, never acks
- **Rule, replacing revision §3.4's "a new batch starts at every ack and at every cold boot":**
  - A new batch key is drawn at every scheduled window, whether or not the upload succeeds.
  - A new key is also drawn on every reset that isn't a deep-sleep wake.
  - An ack never rotates the key.
- **Backstop:** if a batch reaches half the ring's slots, a new key is drawn at once.
- **If the radio can't start** to draw a key, for example under `low_batt_skip`, the device stops writing door slots until it can. The report shows that window as a gap with the cause `no_key`.
- **Uniqueness, restated:** a batch lasts at most one 3-hour window. At the 20-per-minute chatter cap, that is at most about 3,600 door slots plus 3 health rows, well under the ring's roughly 21,500 slots. So no position is written twice under one key. The backstop covers anything unforeseen. This replaces addendum-2's "more than 30 days to wrap" argument.
- **0006 text, added:** "A batch closes at each scheduled window. Its key is never reused after that."

### R6. Install windows start only on battery power, with no USB
- **Rule:** a window starts only on a power-on reset while the unit is `armed`, with `battery_mv` showing cells fitted and no USB host connected (`usb_serial_jtag_is_connected()` is false).
- **Bench card:** `arm-install` is the last bench step, run with the cells out. Don't power the unit again until you're on site.
- **Gates:**
  - Plugging USB into an armed unit with no cells starts no window.
  - An armed unit kept on USB power for 25 h stays `armed`.

## Deferred to the firmware storage spec, with binding requirements

The spec must meet each of these, and each one's gate must pass before the field trial.

- **R2. No record crosses a sector boundary.**
  - **Ring:** 42 slots per 4 KB sector, with the last 64 B left erased. That gives 21,504 slots, about 96 days at 100 opens a day, and meets the 30-day floor up to about 345 opens a day. These ring figures replace addendum-2's.
  - **Key log:** entries are padded to 256 B, 16 per sector.
- **R3. Key-log ring rules:**
  - Each entry is written before its batch's first slot, marker last, and only when that first slot is actually due. A reset loop writes no entries.
  - The key-log head is found by the same erased-run rule as the ring.
  - A key-log sector is never erased while it holds an entry the unsent data still needs, except under the 90-day rule. When the log is full, the device keeps the current key and logs `keylog_full`.
  - **Gate:** after 200 power-on resets in a row, every unsent batch can still be read.
- **R4. Power-cut recovery:**
  - Pointer records carry the position, the key-log index, a counter and a marker. At boot, the record with the higher counter wins. If neither is valid, the device re-uploads from the tail.
  - At boot, a half-erased sector next to the erased run is erased again.
  - The head is the first erased slot after the newest slot with a marker.
  - If a key-log entry is torn, its slots are reported as a gap.
  - **Gate:** add 10 cuts each during a ring erase, a key-log write and a pointer update to the 30-cut freezer gate.
- **R5. Uploads after a missed window:**
  - The pointer record holds the key-log index.
  - Each upload carries the wrapped keys for the slots it covers.
  - Uploads go in fixed-size chunks at the largest size step, oldest first, and the pointer moves forward with each acked chunk.
  - Every install-mode upload is exactly 22 slots, oldest first, and any surplus waits for the next tick.
  - **Gate:** a 7-day outage is drained over the hotspot within a stated number of windows. This extends N21.

## Decision text and corrections

- **T1. 0006 text, added:** "USB accepts only `dump-sealed` and `arm-install`. `dump-sealed` sends sealed slots and wrapped batch keys. `arm-install` is unsigned, works only in bench state, and erases the ring and key log; it can't run again without re-provisioning. No console."
- **T2. N13's disclosure text, replaced:**
  - "A flash dump that also gets past flash encryption shows how many ring slots are in use (roughly the records since the last wrap), the unsent count since the last ack (about 3 h when uploads succeed, longer after an outage), the number of key-log entries (powered time plus resets), the powered-hour count, and torn slots at power cuts."
  - "Of these, only the slot counts depend on door activity, and only in total."
  - "It shows no record times, types, or batch boundaries beyond resets."
  - "A unit that was unpowered may keep its sealed copy past 90 wall days, by however long it was off."
- **T3. Expiry margin:**
  - A key-log sector is erased when its newest entry is older than 90 d × (1 − r) − 2 d − 1 d, measured in powered hours.
  - `r` is the worst raw slow rate of the RC clock, taken from the uncorrected ack offsets in the drift-gate run.
  - The 2 d is one key-log sector's span. The 1 d covers heartbeats lost at resets.
- **0006 90-day line, added:** "On the device, 90 powered days less the margin above. See the consequences."
- **T4. `expired`:** records the powered-hour range of the erased key-log entries, not a slot range. In the report, slots that no key opens, next to an `expired` row, form the `expired` gap. The D1 field list now covers `expired` (powered-hour range), `keylog_full` and `no_key`.
- **T5. W1 worst case, corrected:**
  - A failure after association runs to the 20 s cap, at about 0.61 mAh per try.
  - With every window failing that way, plus hotspot scans: about 4.50 − 2.44 + 4.89 + 1.33 = **8.3 mAh/day**. That gives about 140 days at freezing, still more than 4 times the trial. This replaces addendum-2's 5.2.
  - **Bench:** measure, in the cold, both a failure to associate and a failure after associating.
- **T6. Public page suppression, added to N17:** if a set of totals has exactly one "<3", also suppress that set's next-smallest cell. A suppressed cell can then never be recovered by subtracting from the week's total.
- **T7. Scan rule, added to 0002:**
  - "Scan results other than the configured SSIDs are discarded in RAM and never stored or sent."
  - "`rssi` refers only to the joined access point."
  - "No promiscuous mode and no CSI."
  - This applies to both the trial firmware and the W2 logger.
- **T8. Ageing pulled copies:**
  - The CLI ages each pulled blob by its oldest record once decoded, and deletes `dump-sealed` files once they're imported.
  - R2 can only age blobs by upload time, so a record can stay on R2 up to 90 days after upload. The T3 text says this.
  - A4's backup exclusion covers cloud-synced folders too, not just Time Machine.

## W-A. The W2 Wi-Fi logger

- **Board:** the sacrificial XIAO, before it's locked, running logger firmware committed to the repo. It carries no holder keys, no identity key, and no door sensor.
- **Login:** its own throwaway Wi-Fi login, which the host removes or changes after the gate.
- **What it does:** it sends a fixed-size dummy payload to a test route on the Worker. That route stores nothing but arrival time and success. T7 applies, so no scan list is kept.
- **Readout:** a holder reads the log over USB after the 48 h. The log is deleted once the gate decision is written down. Only the pass or fail result and summary numbers go in the repo, never the raw log.
- **0003 text, added:** "Before a trial, a 48-hour logger with no keys and no door sensor may measure Wi-Fi at the mount spot. It records only joined-AP RSSI, attempt results and association time, and its log is deleted after the gate decision."

## Non-blocking

- **W-1.** The slot payload is 73 B, not 75. A 45–55 B health row still fits.
