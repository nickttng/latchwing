# Review 3, round 01

Reviewer: design-review. Scope: the pick in `verdict.md` (`revision-scrappy.md` plus grafts 1–8 and fixes F1–F4), as amended by `addendum.md` and then `addendum-2.md`. Order of authority: addendum-2 (which replaces B2, B3, B4 and W1 of the addendum), then the addendum, then the verdict, then the revision. I checked it against the README principles, decisions 0001–0006, the brief's hard constraints, the six skills in `skills/`, and every item in `review.md` and `review-2.md`. Date: 2026-11-03.

## Result: needs changes

**Hard constraints: none changed and none failed.** Nothing adds cameras, audio, presence sensing or identification. Nothing publishes raw rows or adds a live door feed. Install-mode uploads stay door-independent. Only holder keys open sealed data. The brief is unedited. Addendum-2 leaves recording the Wi-Fi site fact to the user.

Addendum-2 resolves every blocking item in `review-2.md` (N1–N8) in principle. Both simplifications, the position nonce and the bench-only unsigned `arm-install`, are sound. The remaining defects are in how the new on-flash structures and the upload are specified. Six items must be fixed before firmware is written or the bench gates run (R1–R6). None needs a new mechanism. Each needs one or two sentences plus a gate.

## Techniques touched

All six:
- `access-control`: nonce, key log, crypto-erase, USB inputs, state machine.
- `append-only-events`: binary slots, the `expired` field, the drift rule.
- `store-and-forward`: ring, upload pointer, uploads, W1, the 48-hour gate.
- `time-bucketing`: D5 (N17).
- `enclosure-and-power`: the W1 budget, the partition fit, the W2 gate.
- `event-driven-wake`: unchanged; the chatter cap is used here only to bound the slots per batch.

## Status of review-2.md items

| Item | Status | Why, if not fully resolved |
|---|---|---|
| N1 | Resolved | No batch index, no sector times. Key-log stamps are powered hours. The pointer sits in alternating erased sectors. NVS now holds only state, window count, heartbeat count and the identity key, and none of them is door-derived. This depends on rotation following the schedule (R1). |
| N2 | Resolved | No clock is involved in expiry. The three bench tests cover it. The margin needs a different source (T3). |
| N3 | Resolved | The claim is corrected, and erasing a sector replaces the NVS delete. |
| N4 | Resolved | 96 B slot. Minor arithmetic: the payload is 73 B, not 75 (W-1). Slots straddle sectors (R2). |
| N5 | Resolved | `dump-sealed` includes the key log, and the 0003 text is given. |
| N6 | Resolved by removal | No signature, so no signing key is needed. The 0006 USB text is still missing (T1). |
| N7 | Resolved by removal | Bench-only, one-way state, so there's nothing to replay. |
| N8 | Resolved | Windows are counted at start, and the stop rule is reworded. |
| N9 | Resolved | |
| N10 | Partly | 22 slots is right, but nothing says what happens when a minute holds more (a boot row, a chatter row, or a failed previous tick). See R5. |
| N11 | Resolved | |
| N12 | Resolved | The position nonce replaces the derived nonce. Its uniqueness rests on a rotation rule the revision contradicts (R1). |
| N13 | Partly | The text is added, but two claims are wrong (T2). |
| N14 | Resolved | |
| N15 | Partly | The device has no index, so `expired` can't carry a slot range (T4). |
| N16 | Resolved | |
| N17 | Partly | Two sets of totals for one week, but a single "<3" can be recovered by subtraction (T6). |
| N18 | Resolved | |
| N19 | Partly | The sum uses the old 8 s cap, and the new 20 s cap raises the worst case (T5). |
| N20 | Resolved, with W2 issues | It is now a gate before parts are bought. See the W2 section. |
| N21 | Resolved | Depends on R5 for long outages. |
| N22 | Deferred to the user | Acceptable: changing the skill waits on the 0002 amendment. |
| N23 | Resolved | The fallback is dropped. |
| N24 | Partly | The erased-sector rule is stated. Slots straddling sectors and an interrupted erase break "the single erased run" (R2, R4). |
| New-flow: `arm-install` | Partly | Now unsigned and bench-only, but there is no 0006 text for the USB interface (T1). |
| New-flow: `dump-sealed` | Resolved | 0003 text given. |
| New-flow: signing key | Not applicable | Dropped. |
| New-flow: key log, sector times | Resolved | Sector times are gone. The key log holds only wrapped keys and powered-hour stamps, disclosed in N13 (with T2). |
| A4 | Partly | Pulled copies are aged by pull time, not write time (T8). |
| A5 | Resolved | `arm-install` erases the ring and the key log. |

## Status of review.md items left open by review-2

| Item | Status | Why, if not fully resolved |
|---|---|---|
| B2 | Resolved | One slot size, nothing in the clear but the marker, no index, and the dump can be opened. Remaining problems are in R1, R2 and R4. |
| B3 | Resolved | Crypto-erase by powered hours. The margin's source is in T3. |
| B4 | Resolved | One-way state machine. The USB power-on hole is in R6. |
| D5 / A1 | Resolved except T6 | |
| S5 | Resolved | Through N5. |
| T2 (skill) | Deferred | N22. |

## Blocking: fix before firmware and bench gates

- **R1. Rotation rule: the nonce argument assumes "a batch lasts at most 3 h", but the revision rotates only on acks.**
  - **Problem:** revision §3.4 says "A new batch starts at every ack and at every cold boot." If Wi-Fi is down, one batch key then lives until the next ack. At the chatter cap (about 2,600 slots a day) the ring wraps in about 8 days, so a long outage reuses (key, position) pairs. Under ChaCha20-Poly1305, a reused pair leaks the XOR of the two plaintexts and allows forgery. It also keeps one key in RTC memory for the whole outage, which widens 0006's exposed window. Addendum-2 says "on the fixed schedule" but never replaces the revision's sentence.
  - **Fix:** "A new batch key is drawn at every scheduled window, whether or not the upload succeeds, and on every reset that isn't a deep-sleep wake. An ack doesn't rotate the key." Add a backstop: a batch that reaches half the ring's slots draws a new key at once. If the radio can't start (for example under `low_batt_skip`), the device stops writing and the report shows a gap.
  - **Better argument:** base uniqueness on the per-batch bound (at most about 3,600 door slots plus 3 health rows in 3 h at the 20-per-minute cap, against about 21,500 slots), not on "more than 30 days to wrap". That claim is false under chatter and above about 350 opens a day.
- **R2. 96-byte slots don't divide a 4,096-byte sector (42.67 per sector), so slots straddle erase boundaries.**
  - **Problem:** when the ring erases the sector ahead of the head, the slot that straddles into the next sector loses its first part but keeps its marker. The device's check then counts it as in use and not `torn`, the report finds a slot it can't authenticate, and the `overwritten` range is off by one on every wrap. The same applies to 246-byte key-log entries (3 × 80 B wrapped key, plus stamp and marker).
  - **Ring fix:** pack 42 slots per sector and leave the last 64 B erased, so no slot crosses a sector. That gives 512 × 42 = 21,504 slots: about 96 days at 100 opens a day, with the 30-day floor met up to about 345 opens a day.
  - **Key-log fix:** pad entries to 256 B, 16 per sector, which is 2 days at 8 batches a day.
  - Restate the ring numbers in B2.
- **R3. The key log has no ring rules, and if it fills it can crypto-erase unsent data.**
  - **Problem:** 256 KB holds 1,024 entries, with 720 needed for 90 days. Every non-deep-sleep reset adds an entry, so a cold brownout loop (for example RF start at boot sagging the cells) could fill the log in hours. Nothing says how the log wraps, or what happens when it is full. The obvious choice, erasing the oldest sector, destroys the keys of batches that haven't uploaded yet.
  - **Fix:**
    - Write each entry, marker last, before the batch's first slot.
    - Write the entry only when the batch's first slot is due, and defer the reset-reason row to the next hourly row, so a reset loop writes no entries.
    - Find the key-log head with the same erased-run rule as the ring.
    - Never erase a sector holding an entry at or after the upload pointer's key index (R5), except under the 90-day rule. When full, keep the current key and log `keylog_full`.
  - **Gate:** 200 back-to-back power-on resets leave every unsent batch readable.
- **R4. Power-cut recovery is unspecified for the three new flash structures.**
  - **Problem:** the verdict's power-cut gate cuts power only mid-write. A cut during a sector erase can leave a sector half-erased, giving more than one erased run, which breaks the N24 rule. A cut between "write new pointer sector" and "erase old" leaves two valid pointers. A torn key-log entry makes its batch unreadable.
  - **Fix:**
    - Each pointer record carries the position, the key-log index, a counter and a marker written last. Boot takes the valid record with the higher counter. If neither is valid, it re-uploads from the ring tail.
    - At boot, any sector that is neither fully erased nor cleanly written, and that sits next to the erased run, is erased again before writing.
    - The head is the first erased slot after the newest slot with a marker.
    - A torn key-log entry is logged, and its slots are reported as a gap.
  - **Gate:** add 10 cuts each during a ring-sector erase, a key-log write and a pointer update to the 30-cut freezer gate.
- **R5. The upload's contents are undefined once a window fails, and a large backlog may never fit W1's 20 s cap.**
  - **Problem:** B2 says "each upload is the batch's slots plus its start position", in the singular. After a missed window the unsent range spans several batches. With no index, the device can't tell which wrapped keys to include. W1 also gives no upload size limit, so a backlog after weeks offline (the hotspot-rescue case, hundreds of KB) can fail every 20 s attempt for good, and the ring then overwrites it.
  - **Fix:**
    - The pointer record holds the key-log index of the batch containing the pointer (R4).
    - Each upload carries the wrapped keys from that index to the newest entry it covers.
    - Each upload carries at most a fixed number of slots, the largest size step, oldest first, and the pointer advances on each acked chunk.
    - Every install-mode upload is exactly 22 slots, oldest first, and any surplus waits for the next tick. That keeps the size fixed, as N10 intends.
  - **Gate:** extend the N21 gate to a 7-day outage drained over the hotspot within the stated number of windows.
- **R6. A USB power-on in `armed` state starts an install window off site.**
  - **Problem:** the firmware can't tell "cells inserted" from "USB plugged in". Both are power-on resets. If a holder plugs in an armed unit again (for example to check it or run `dump-sealed`), that uses up a window and starts the 24-powered-hour clock. Enough of that and the unit reaches `field` before install day, step 5 of the install card fails, and the board has to be replaced. This is the failure from review.md B4, by another route. If `arm-install` is run with cells fitted, the unit also stays powered and armed in transit.
  - **Fix:**
    - Start a window only on a power-on reset with `battery_mv` showing cells fitted and no USB host connected (`usb_serial_jtag_is_connected()`, or the C3 equivalent).
    - State on the bench card: run `arm-install` with cells out, as the last bench step, and don't power the unit again until site.
  - **Gates:**
    - USB plugged into an armed unit with no cells starts no window.
    - An armed unit powered on USB for 25 h stays `armed`.

## Fix in text before the decisions are accepted

- **T1. No 0006 text for the USB interface.** **Fix:** add "USB accepts only `dump-sealed`, which sends sealed slots and wrapped batch keys, and `arm-install`. `arm-install` is unsigned, accepted only in bench state, erases the ring and the key log, and can't run again without a new board. No console."
- **T2. N13's disclosure text has two wrong claims.**
  - "At most about 3 h of records" is wrong. The unsent count is everything since the last ack, which is days after an outage.
  - "Key-log entries (powered time only)" also shows the number of resets. Torn slots also mark power cuts, which are some batch boundaries.
  - **Fix:** reword to "the unsent count since the last ack (about 3 h when uploads succeed), the number of key-log entries (powered time plus resets), and torn slots at power cuts. None of these depends on door activity except the unsent count."
- **T3. The expiry margin comes from the wrong measurement.**
  - The drift gate measures leftover error after interpolation, not the raw RC rate. A slow RC clock undercounts powered hours, so the device copy outlives 90 days. Each reset also loses up to 1 h of heartbeat, and erasing a whole sector keeps entries up to 2 days past the cutoff (R2).
  - **Fix:** erase when the newest entry in the sector is older than 90 d × (1 − worst raw slow rate) − one sector's span (2 d) − 1 d. Take the worst raw rate from the uncorrected ack offsets of the drift-gate run.
- **T4. `expired` can't carry a slot range,** because the device doesn't know which slots an erased key covered. **Fix:**
  - `expired` records the powered-hour range of the erased entries.
  - In the report, slots that no key opens, next to an `expired` row, form the `expired` gap.
  - Change the D1/N15 wording to match.
- **T5. The W1 worst case is understated.** A failure after association (AP up, upstream or Worker down) runs to the 20 s cap: about 0.61 mAh per try. **Fix:** worst case about 4.50 − 2.44 + 4.89 + 1.33 = 8.3 mAh/day, about 140 days at freezing, still more than 4 times the trial. Measure both kinds of failure cold.
- **T6. A single "<3" can be recovered by subtraction.**
  - The 7 day totals and the 24 hour totals both add up to the same weekly total. If exactly one cell in either set is suppressed, it equals that total minus the cells shown.
  - **Fix:** if a set has exactly one "<3", also suppress its next-smallest cell, or show no total and round both sets to the nearest 5.
- **T7. Scan results are not ruled out of storage.**
  - Passive scans (graft 6, and the W2 logger) return every nearby SSID and BSSID, including personal hotspots with people's names in them.
  - **Fix:** add to 0002 or 0006: "Scan results other than the configured SSIDs are discarded in RAM and never stored or sent. `rssi` is the joined AP's only. No promiscuous mode or CSI."
- **T8. A4 ages pulled blobs by pull time.**
  - A `dump-sealed` file or hotspot rescue pulled on day 80 holds day-1 records. Deleting it 90 days after the pull breaks T3's "90 days after they were written".
  - **Fix:** the CLI ages each blob by its oldest record once decoded, and deletes `dump-sealed` files once imported.
  - R2 can only age by upload time. Say so in the T3 text.
  - A4's backup exclusion should also name cloud-synced folders, not just Time Machine.

## W2, the site Wi-Fi logger

- **It adds no data about people as written.** It logs RSSI to the joined AP, the result, and the time to associate, with no door sensor. T7 must also apply, so that no scan list is kept.
- **W-A. Fix before W2 runs: the logger's board, login and readout are unspecified.** This is a new way to move and read data, so it needs a short note in 0003, or a one-line decision.
  - If it runs before the sacrificial lock (W2 says before boxes are bought), it is an unlocked board in public for 48 h with the Wi-Fi login in plaintext flash.
  - If it runs after the lock, it can't take new firmware, because release-mode flash encryption has no OTA.
  - **Fix:**
    - Run it before the lock, with logger firmware committed to the repo.
    - Use its own throwaway Wi-Fi login, removed after the gate.
    - Send a fixed-size dummy payload to a test route.
    - Read the log over USB by a holder, and delete it after the gate decision.

## New fields, sensors and data flows

- **The W2 logger** (W-A) and its readout over USB: needs the note in W-A.
- **The powered-hour counter and key-log stamps:** device state with no door data, disclosed by N13 (with T2). No decision record is needed beyond that text.
- **The pointer record** (position, key index, counter): disclosed by N13 (with T2).
- **`keylog_full`** (R3) and the reworded `expired` (T4) go in the D1 health-field list.
- **No new sensors and no live feeds.** USB inputs need T1.

## Departures, checked

- **Have replacement text:** B1 (0003), N5 (0003), N9 (0003), N14 (0004), N15 (0002), N17 (0004), N13 (0006 consequences, including the device copy outliving 90 wall days while unpowered).
- **Still missing:**
  - 0006 USB text (T1).
  - The rotation rule (R1). It is also a 0006 point: "a batch closes at each scheduled window".
  - The scan rule (T7).
  - Suggestion: also add "on the device, 90 powered days, see consequences" to 0006's "deleted after 90 days" line, so the decision line itself matches.
- **Skill departures for the user to accept:**
  - binary slots (`append-only-events`);
  - one try per window (`store-and-forward`);
  - the drift rule (N22).
  - Not a departure: the position nonce is a counter, as `access-control` asks, and stays with libsodium primitives.

## Non-blocking

- **W-1.** The payload is 96 − 16 − 2 − 1 − 4 = 73 B, not 75. A 45–55 B health row still fits.
- **W-2.** "Only a full re-provisioning returns a unit to bench": with release-mode flash encryption and no OTA, that means a new board (critique-scrappy #2). Say so.
- **W-3.** RTC fast memory survives software and watchdog resets on the C3. "Discards the batch key" must be an explicit wipe at boot, keyed to `esp_reset_reason()`. Add a gate: a panic reset produces a new key-log entry.
- **W-4.** The trial-decryption walker must allow for batches with zero slots, and for erased keys. Pass `crypto_kdf_derive_from_key` a fixed subkey ID along with the 8-byte context `"ringslot"`.
- **W-5.** Confirm that the 4 MB partition fit includes the Secure Boot v2 bootloader offset: 2 MB ring, 256 KB key log, 8 KB pointer, NVS, and an app of about 1.3 MB.
- **W-6.** W2's "Cellular is excluded" is a user preference, not a brief constraint. Keep it labelled that way.
