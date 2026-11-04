# Addendum 2, round 01

This answers `review-2.md`. **It replaces B2, B3, B4 and W1 in `addendum.md`.** Everything else in `addendum.md` still stands, except where an N-fix below rewords it. Order of authority: addendum-2, then addendum, then verdict, then revision. Date: 2026-11-03.

Hard constraints: unchanged. Transport: unchanged. Cellular is ruled out, and Wi-Fi at the fridge is spotty, with detailed connection data pending (see W2).

## Two simplifications

1. **The nonce is the slot's position in the ring.** The access-control skill already asks for a counter as the nonce, and the position is one. No nonce is stored, so a slot has more room for its payload, and there's no derivation scheme to get wrong (N4, N12).
2. **`arm-install` is unsigned and works only in bench state.** B1 makes install-mode uploads independent of the door, so an unauthorized arm reveals nothing; it can only cost battery. A one-way state machine then does what a signature was meant to do, with no new holder secret and nothing to replay (N6, N7).

## B2 (replaced). Ring slots

- **Slot layout:** 96 B: `[ChaCha20-Poly1305-IETF ciphertext ‖ 16 B tag][2 B fixed marker]`. The plaintext is `type (1 B) ‖ seq (4 B) ‖ payload`, padded. That leaves 75 B for the payload, and a packed health row is about 45–55 B. Every record, door or health, takes exactly one slot.
- **Encoding:** slots are packed binary. This departs from the `append-only-events` skill's "JSON lines" rule; the report CLI decodes them.
- **Key and nonce:**
  - The per-batch AEAD key is `crypto_kdf_derive_from_key(batch_key, "ringslot")`.
  - The nonce is the slot's absolute position in the ring, as 96-bit big-endian.
  - **Why it never repeats:** a position is written once per batch key. A batch lasts at most 3 h, and the ring takes more than 30 days to wrap. Torn slots are skipped and never rewritten. A reset that isn't a deep-sleep wake discards the batch key and draws a new one (F-rule from 0006), and positions carry on from the head.
- **No batch index anywhere.** The key log holds only `[wrapped batch key][powered-hour stamp]`, one entry per batch, on the fixed schedule. A holder finds each slot's batch by trial decryption, walking slots and keys in order. That is about 720 keys × 22k slots, which takes seconds on a laptop.
- **Uploads:** each upload is the batch's slots plus its start position, inside the existing sealed, padded batch envelope. The server never sees positions or slot counts.
- **Head scan:** the ring always keeps at least one erased sector ahead of the head. A slot is in use if any byte isn't 0xFF. The head is the start of the single erased run, counted around the ring. A slot that is in use but has no marker is `torn`.
- **Upload pointer:** kept in two alternating dedicated sectors, with the old one erased on each update. It is not kept in NVS, because NVS leaves old values in flash.
- **Partitions:**
  - The ring grows to 2 MB, which is about 21,800 slots. That's about 97 days at 100 opens a day, and it meets 0003's 30-day floor up to about 350 opens a day.
  - The key log is 256 KB, enough for 90 days × 8 batches × about 240 B.
  - **Bench item:** confirm the partition table fits in 4 MB, with a single app partition and no OTA.
- **Fallback dropped:** the "rotate every 6 hours" fallback is gone (N23). If the key log doesn't fit, grow the partition.
- **Verdict §7 item 6:** the "recheck ring size" threshold drops from 1,000 opens a day to 300.

## B3 (replaced). The 90-day rule: crypto-erase by powered hours

- **Age is measured in powered hours, not clock time.** The device counts heartbeats in NVS. That count reveals only how long the device has been powered on.
- **Erasing:** when every entry in a key-log sector is older than 90 days minus a drift margin, the device erases that whole sector. The margin comes from the bench drift gate, at about 1 day. Without their batch keys, the device's ring slots for those batches can't be read by anyone. The ring itself overwrites as normal.
- **No clock is involved.** An unset clock or a server-set clock can't expire anything (N2).
- **Bench tests:**
  - A cold boot, with batches written before `clock_set`, expires nothing.
  - An ack with a time 200 days ahead expires nothing.
  - A key-log sector whose stamps are backdated past the limit gets erased.
- **Remaining gap:** powered hours undercount wall time while the device is unpowered. The device's sealed copy can outlive 90 wall days by however long the device was off. This is disclosed in N13.
- **N3, corrected claim:** key erasure makes only the device's copy unreadable. Uploaded, stored and pulled copies carry their own wrapped keys. They are covered by R2's 90-day lifecycle and by T3, which the CLI enforces under A4.
- **N14:** records that expire before they upload leave a window with no data. The report shows it as a gap with the cause `expired`, and `expired` joins the 0004 gap list.

## B4 (replaced). What arms install mode

- **States** are one-way and kept in NVS: `bench` → `armed` → `field`. Only a full re-provisioning, which wipes all keys, returns a unit to `bench`.
- **Bench:** install mode never runs. The bench gates run in this state.
- **`latchwing arm-install`** goes over USB and is accepted only in `bench`. It does three things:
  - erases the ring and the key log, so bench records never reach the trial (A5);
  - starts a new batch;
  - sets `armed`.

  In `armed` or `field`, the command is ignored.
  - **No signature, and why that's safe:** B1 makes install uploads door-independent, so arming reveals nothing. A unit is only in bench state before install day.
- **Starting a window:** only on a power-on reset, meaning cells inserted, while the unit is `armed`. A brownout, watchdog or panic reset never starts one.
- **Counting windows:** each window is counted when it starts, written to NVS before its first upload.
- **Leaving `armed`:** the unit moves to `field` on the first signed ack in an install window, after 3 windows, or 24 powered hours after the first window started, whichever comes first.
- **Verdict §8 stop rule, reworded:** "Install mode starting while the unit is in `field` state, which means after an ack, after 3 windows, or more than 24 h after the first window."
- **Gates:**
  - Install mode runs on site, after the bench gates.
  - A brownout reset in `armed` starts no window.
  - In `field`, 5 power cycles start no window.
  - `arm-install` is ignored in `armed` and in `field`.

## Fixes to wording and decision text

- **N5. `dump-sealed`** sends the sealed slots and the key log, whose contents are wrapped to holders. **0003 transport text:** "a USB dump of sealed frames and their wrapped batch keys."
- **N8.** Covered by B4 above.
- **N9. 0003 trigger text, replacing addendum B1's sentence:**
  - "Uploads run on the fixed schedule."
  - "The other triggers are the 50-percent guard, the install-mode timer, and an hourly hotspot try after a missed window."
  - "Apart from the guard, timing never depends on door activity."
- **N10. Install padding:** every install-mode upload is padded to one fixed size, 22 slots. That holds a full minute at the chatter cap: 20 door events and 2 health rows.
- **N11. Install-check decrypt:** the install check decrypts on a holder laptop that meets F3, never on a phone. The holder can be off site, on a call with the volunteer.
- **N13. 0006 consequences, added:**
  - "A flash dump that also gets past flash encryption shows how many ring slots are in use (roughly the records since the last wrap), the unsent count at the current upload pointer (at most about 3 h of records), how many key-log entries there are (powered time only), and the powered-hour count."
  - "It shows no record times, types, or batch boundaries."
  - "A unit that was unpowered may keep its sealed copy past 90 wall days, by however long it was off."
- **N15. D1 list:** `overwritten` and `expired` join the door-derived fields. **0002 text:** "the record type and sequence number sit in the sealed envelope."
- **N16. D4:** add "sent only to named holders."
- **N17. D5, replaced:**
  - "Each weekly public page shows, for that completed week only, two sets of totals: by day of the week, and by hour of the day across the week. Counts under 3 show as '<3'."
  - "No page publishes per-cell totals, or totals that add up across weeks."
  - "Pages are posted by hand by a holder and never updated automatically."
  - The restock coordinator's working page goes to a shared folder, not to the public. It is unchanged.
- **N22. Skill update:** once the 0002 amendment is accepted, the `append-only-events` skill's drift rule gets the F1 wording. That is the user's call, along with the decisions.
- **Skill departures for the user to accept:**
  - binary slots instead of JSON lines (`append-only-events`);
  - W1's single try per window instead of "retry with backoff" (`store-and-forward`). Fixed timing is better for privacy.

## W1 (replaced). The cap on upload attempts

- **Caps:**
  - 8 s for association and DHCP;
  - 20 s in total per attempt, including transfer and ack;
  - one attempt per window.

  The same caps apply to hotspot tries and guard uploads. A failure logs `upload_fail`.
- **Worst case** (N19), with every window failing plus the hourly hotspot scans: about 4.50 − 2.44 + 1.78 + 1.33 = **5.2 mAh/day**. That gives a life of about 220 days at freezing, still more than 7 times the trial. These are estimates.
- **Bench:** measure a failed attempt cold, at weak RSSI.
- **N21, added gate:** cut the network for 48 h with W1 on. Pass when `upload_fail` is logged once per window, no record is lost, and everything uploads within 2 windows after the network comes back.

## W2. Site Wi-Fi gate (N20)

Spotty Wi-Fi doesn't yet pass check-first item 1, so the detailed connection data you asked for becomes a gate.

- **Logger:** before buying boxes and mounting parts, run the sacrificial XIAO as a Wi-Fi logger at the exact mount spot for 48 h, with the host's permission. It runs the real W1 attempt on the 3-hour schedule, plus a timed attempt every 15 minutes.
- **What it logs:** only RSSI, the attempt result, and the time to associate. There is no door sensor attached.
- **Pass:** at least one successful attempt in every 12-hour span (the silence-email window), and at least 50% of the 3-hour windows succeed.
- **Fail:** try the best spot within 1 m. If that fails too, add an extender, which needs the host's outlet, or move to round 02 on how data gets out. Cellular is excluded.
- **Brief:** this site fact could be recorded in the brief once it's known. That is the user's call.

## Advisory

- **A4.** On every run, the `latchwing` CLI deletes pulled blobs older than 90 days. Its data folder is left out of laptop backups, such as Time Machine.
- **A5.** Covered by B4 above: `arm-install` erases bench records.
