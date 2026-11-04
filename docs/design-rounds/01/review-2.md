# Review 2, round 01

Reviewer: design-review. Scope: the pick in `verdict.md` (`revision-scrappy.md` plus grafts 1–8 and fixes F1–F4), as amended by `addendum.md`. Where they conflict, the addendum wins. I checked it against the README principles, decisions 0001–0006, the brief's hard constraints, the skills in `skills/`, and every item in `review.md`. Date: 2026-11-03.

## Result: needs changes

**Hard constraints: none changed and none failed.** Nothing adds cameras, audio, presence sensing or identification. Nothing publishes raw rows or adds a live door feed. Only holder keys open sealed data. The $50 budget the addendum goes over (A2) is a trial default in the brief, not a hard constraint.

The addendum resolves most of `review.md`. Its new mechanisms add seven problems that must be fixed before firmware is written or the bench gates run (N1–N8 below):
- The B2 key store and the B3 sector times hold door-derived counts under a key the device holds.
- B3 expiry trusts a clock that may be unset, or set by the server.
- The 64-byte slot can't hold a health row.
- A USB dump can no longer be opened.
- `arm-install` can't be signed with the holders' existing keys, and a signed command could be replayed or reused.

## Techniques touched

All six: `event-driven-wake` (S3), `append-only-events` (B2, D1, D2, T1, T2), `store-and-forward` (B1, B2, B3, W1), `time-bucketing` (D5), `enclosure-and-power` (S1, S2, S6, W1), `access-control` (B2, B3, B4, T3, S4, S5).

## Status of review.md items

| Item | Status | Why, if not fully resolved |
|---|---|---|
| B1 | Resolved | 60 s timer, padded, never on a door change. Wording fixes in N9 and N10. |
| B2 | Partly | The plaintext `len`/`seq` header is gone: one slot size, `seq` and type inside the AEAD, and a marker. But the batch-start index moved into a store guarded by a device-held key (N1). The slot is too small (N4). Dumps can't be opened (N5). The nonce is under-specified (N12). |
| B3 | Not resolved as written | Per-sector times fix "the device can't read record ages". But sectors filled before `clock_set` carry unset-clock times, and a server-set clock can expire everything (N2). The key-deletion claim is wrong (N3). Expired windows must still show as gaps (N14). Sector times also leak (N1). |
| B4 | Resolved, with new problems | What arms install mode and what uses it up is now stated. But the holder keys can't sign (N6), the command can be replayed or reused (N7), and the retry windows clash with a section 8 stop rule (N8). |
| D1 | Resolved | Minor: N15. |
| D2 | Resolved | The ESP-IDF ADC check stays a bench item. |
| D3 | Resolved | |
| D4 | Resolved | Minor: name the recipients (N16). |
| D5 | Partly | Hand-posted, completed weeks only, "<3". But weekday-by-hour for a single week is the same as date-and-hour cells, and weekly totals that add up over time can be subtracted to recover small counts (N17). |
| T1 | Resolved | |
| T2 | Resolved | The skill still carries the old 5-minute rule (N22). |
| T3 | Resolved | Advisory on enforcement and backups (A4). |
| S1–S4, S6 | Resolved | |
| S5 | Resolved, depends on N5 | A `dump-sealed` copy is only useful if it includes the wrapped keys. |
| A1 | Partly | Settled through D5. See N17. My original suggestion had the same flaw. |
| A2, A3 | Resolved | Holder signing keys (N6) also stay out of the repo. |

## Blocking: fix before firmware and bench gates

- **N1. B2 and B3 put door-derived counts under a key the device holds.** The "flash-encrypted store" keeps three things:
  - the batch-start slot for every 3-hour batch, which gives records per batch;
  - each sector's fill time, which gives an activity timeline at sector granularity;
  - NVS's old pointer values. NVS marks updated entries as erased but leaves them in flash until the page is erased; confirm this in ESP-IDF.

  Only flash or NVS encryption protects this, with a key in the chip's eFuses. That contradicts D1's rule that door-derived data is "sealed, readable only by named holders". It also contradicts 0006's model, in which the device holds no key that reads door data. It also means the addendum's "what a dump still shows" is incomplete, and "flash-encrypted NVS" is not accurate: NVS uses NVS encryption, and per ESP-IDF its page headers and entry-state bitmaps stay in the clear.

  **Fix:**
  - Store no batch-start index. A holder finds each slot's batch by trial decryption against the batch keys, using the slot's stored nonce. That is about 720 keys × 25k slots, which takes seconds on a laptop.
  - A key-log entry holds only the wrapped key and its creation stamp. Batches start on the fixed schedule, so the stamp leaks nothing about doors.
  - Expire data by erasing key-log sectors past their age (crypto-erasure), and let the ring overwrite as normal.
  - Keep the upload pointer in two alternating dedicated sectors, erased on each update.
  - Alternatively, keep the design and add 0006 consequence text naming exactly what a dump that also defeats flash encryption shows.

- **N2. B3 expiry trusts a clock that may be unset or server-set.**
  - Sectors that fill before `clock_set` after a cold boot are stamped from an unset clock. They are then erased the moment the clock is set.
  - A breached server can sign a first ack after a cold boot that sets the clock 100 days ahead. That expires the whole ring, which reopens the R2 "an ack deletes data" problem by another route.
  - Under F1 the RC clock is never stepped again, so over 90 days it drifts by its full error.

  **Fix:** measure age in powered hours, kept as an NVS count of hourly heartbeats, and use no clock. Expire at 90 days minus a margin taken from the measured RC drift. Add two bench tests: a cold boot with sectors filled before `clock_set` expires nothing, and an ack 200 days ahead expires nothing.

- **N3. B3's claim that "without its key, any surviving copy can't be read" is false.** Each upload, the R2 blob and every pulled copy carries its own wrapped-key header, so holders can still read those copies. Also, an NVS delete only marks the entry; it doesn't erase it. **Fix:** say that key deletion makes only the device's copy unreadable, and that R2's lifecycle and T3 cover the other copies. Delete keys by sector erase, not NVS delete.

- **N4. A 64-byte slot can't hold a health row.** 64 B minus the 12 B nonce, 16 B tag, 2 B marker, 1 B type and 4 B `seq` leaves 29 B for the payload. The D1 health row is about 45–55 B packed as binary, and hundreds of bytes as the JSON lines that `append-only-events` requires. **Fix:**
  - Size the slot from the largest packed health row (likely 96 or 128 B), or split each health row across 2 slots.
  - Record binary packing as a departure from the skill's "JSON lines" rule.
  - Recompute ring life. At 128 B, 1.5 MB holds 12,288 slots, and 0003's 30-day floor fails above about 190 opens a day.
  - Grow the ring partition (4 MB flash, no OTA), and lower the "recheck ring size" threshold in verdict §7 item 6 to match.

- **N5. A USB `dump-sealed` can no longer be opened.** The wrapped keys now live outside the ring, and `dump-sealed` streams "sealed frames and nothing else". That breaks the USB transport, the 80-day recovery path and S5. **Fix:** `dump-sealed` also streams the wrapped keys, which are sealed to holders already. Change the 0003 transport text to "a USB dump of sealed frames and their wrapped batch keys".

- **N6. `arm-install` can't be signed with the holder keys that exist.** Holder keys are X25519, used with `crypto_box_seal`, and they can't make signatures. **Fix:**
  - Give each holder a separate Ed25519 signing key pair, and build their public keys into the signed firmware.
  - Retire a signing key along with its holder.
  - Add to 0006: "Holders also hold a signing key, used only to arm install mode."

- **N7. A signed `arm-install` can be replayed, and nothing limits it to once per provisioning.** The command names no device and carries no counter, so a captured copy can re-arm any unit. Nothing stops a second arm after install mode is used up, which contradicts graft 1 and the B1 0003 text ("armed once per provisioning"). **Fix:** sign over "arm-install ‖ device identity public key ‖ provisioning ID". Accept the command once per provisioning, with an NVS flag `install_used` that blocks any later arm. If holders should be able to re-arm, reword graft 1 and 0003 to say so.

- **N8. The B4 retry windows clash with a section 8 stop rule.** If the first install window gets no ack, an unattended brownout later restarts install mode in the field. Verdict §8 says "install mode re-arming in the field" stops the trial at once. If a window is counted only when it ends, repeated brownouts give unlimited windows. **Fix:** count each window when it starts, with an NVS write before the first upload. Reword the stop rule to "install mode starting after an ack, or more than 3 times".

## Fix in text before the decisions are accepted

- **N9. Install mode now adds a trigger that conflicts with the 0003 wording.** The B1 0003 sentence "Upload timing never depends on door activity" contradicts the 50-percent guard. The verdict's "the only other trigger" now excludes install mode. **Fix:** "Uploads run on the fixed schedule. The other triggers are the 50-percent guard, the install-mode timer, and an hourly hotspot try after a missed window. Apart from the guard, timing never depends on door activity."
- **N10. B1 install padding is ambiguous.** Normal batches are padded to a set of size steps, so "the same size step as a normal batch" doesn't say which one. **Fix:** pad every install-mode upload to one fixed size large enough for a full minute at the chatter cap (20 wakes/min, so 20 door events and 2 health rows).
- **N11. B1 lets a holder decrypt on a phone.** A private key on a phone is not covered by 0006 or F3. **Fix:** decrypt on the laptop only, or add phone rules: passphrase-protected key file, device encryption, and the key removed after install.
- **N12. The B2 nonce is under-specified, and its stated purpose is wrong.** Specify the construction with libsodium calls:
  - derive the encryption and nonce subkeys with `crypto_kdf_derive_from_key`;
  - make the nonce a keyed BLAKE2b of the counter, truncated to 12 B for IETF ChaCha20-Poly1305;
  - increment the counter before every encryption;
  - keep the key, the subkeys and the counter in the same RTC memory, and discard all of them on any reset that isn't a deep-sleep wake.

  Also correct the stated purpose. A derived nonce hides batch boundaries, which a plain counter would show by restarting at 0. It doesn't hide write order, because slot position already shows that.
- **N13. B2's dump disclosure has no decision text.** **Fix:** add to the 0006 consequences: "A dump shows how many ring slots are in use, roughly the total records since the last wrap. It shows no times, record types, or batch boundaries."
- **N14. B3 reports `expired` data "not as a gap", which 0004 forbids for unsent records.** Records that expire before upload leave a window with no data, and 0004 requires that window to be shown as a gap. **Fix:** show it as a gap with the cause `expired`, and add that cause to the 0004 gap list.
- **N15. D1 misses two door-derived fields and the record type.** `overwritten` and `expired` carry slot ranges, which are record counts, so they belong in the door-derived list. **Fix:** add them to the list, and change the 0002 text to "the record type and sequence number sit in the sealed envelope".
- **N16. D4 doesn't say who the email goes to.** **Fix:** add "sent only to named holders".
- **N17. The D5 page can still show single visits.** For one completed week, weekday-by-hour cells are the same as date-and-hour cells. Totals that add up across weeks can be subtracted from each other, which defeats "<3". **Fix:** each weekly page publishes only two sets of totals for that week: day-of-week totals and hour-of-day totals across the week, each with "<3". It never publishes per-cell totals that add up across weeks.
- **N22. The `append-only-events` skill still has the old drift rule.** It still says to flag drift over 5 minutes, which T2 removes. **Fix:** once the 0002 amendment is accepted, update the skill to the F1 rule: interpolate between offsets and flag `clock_suspect`.
- **N23. The B2 fallback "rotate the batch key every 6 hours" changes 0006.** Separating key rotation from acks keeps already-uploaded records' key in RAM longer, which widens the exposed window in 0006. **Fix:** grow the partition instead. If rotation is used anyway, say so in 0006.
- **N24. The B2 head scan is only correct if an erased gap always exists.** **Fix:** state that the ring always keeps at least one erased sector ahead of the head, and that the head is the start of the single erased run, counted around the ring.

## W1, the upload cap

- **N18. The 8-second timeout is shorter than the 10 s budgeted for a normal upload.** In cold weather with weak signal, it could cut off uploads that would have succeeded. **Fix:** apply 8 s to association and DHCP only, with a separate cap for the transfer and ack (about 20 s in total). Apply the same cap to hotspot and guard attempts. Measure it cold at weak RSSI.
- **N19. The W1 budget double counts and leaves out the hotspot.** A failed attempt replaces the upload's 2.44 mAh rather than adding to it, and the hourly hotspot scans (24 × 2 s at about 100 mA, about 1.33 mAh) are missing. The worst case is about 4.50 − 2.44 + 1.78 + 1.33 = 5.2 mAh/day, so the 6.3 figure is conservative. **Fix:** restate the worst case with this sum.
- **N20. "Iffy" Wi-Fi does not pass check-first item 1 in verdict §7.** W1 caps the battery cost, but it doesn't show that the trial can meet "no site visit". **Fix:** before buying parts, log 48 h of timed connection attempts from the mount spot and set a pass rate. The user may want to record this site fact in the brief.
- **N21. The `store-and-forward` check is not among the bench gates.** The skill requires: "Cut the network for 48 hours and confirm no events are lost." **Fix:** add it as a gate, run with W1 on, and confirm that `upload_fail` is logged once per window and that everything uploads afterwards.

## New fields, sensors and data flows without decision text

- `arm-install` is a new input over USB (N6, N7). **Fix:** add to 0006: "USB accepts only `dump-sealed`, which sends sealed frames and wrapped keys, and a holder-signed, device-bound `arm-install`, once per provisioning. No console."
- `dump-sealed` now sends more than sealed frames (N5). This needs the 0003 text change in N5.
- The holder signing key is a new kind of holder secret (N6). It needs the 0006 text in N6.
- The key log and the per-sector times are a new store of door-derived metadata (N1). They need either the redesign or the 0006 text in N1.
- `upload_fail`, `expired` and `temp_c` are covered by D1 and D2. There are no new live feeds. The install-mode LED is local only.

## Departures, checked

- **Have replacement text:** B1 (0003), D1 and D2 (0002), D3 (0003), D4 (0006), D5 (0004), T1 and T2 (0002), T3 and S4 (0006).
- **Missing text:** the USB interface and the signing key (0006); the `dump-sealed` contents (0003); the record type in the envelope (0002); expired data as a gap (0004); what a dump shows (0006). The fixes are in N5, N6, N13, N14, N15 and the section above.
- **Skill-level departures to accept or write into the skills:**
  - binary slots instead of JSON lines (`append-only-events`);
  - a derived nonce instead of "a counter as the nonce" (`access-control`, still with library primitives only);
  - the 5-minute drift flag replaced (N22);
  - W1's no-retry-until-the-next-window instead of "retry with backoff" (`store-and-forward`), which is fine because fixed timing is better for privacy.

## Advisory

- **A4. T3 needs enforcing on holder laptops.** **Fix:** the `latchwing` CLI deletes pulled blobs older than 90 days on every run. Keep its data folder out of laptop backups such as Time Machine, or the backups break "wherever they are".
- **A5. Bench-state records stay in the trial unit's ring.** **Fix:** `arm-install` starts a new batch, and reports drop every record from before install day.
