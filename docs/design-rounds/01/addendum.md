# Addendum, round 01

This addendum answers `review.md`. It changes the pick in `verdict.md` only where it says so. The pick is still Scrappy's revised design with grafts 1–8 and fixes F1–F4. Where an item here conflicts with the verdict or the revision, this file wins. Date: 2026-11-03.

Nothing here changes the brief's hard constraints, and nothing here changes how data leaves the device. Everything below holds whichever path the site visit settles on. Cellular is ruled out. Wi-Fi at the fridge is spotty, and detailed connection data is pending the site visit.

## Blocking items

### B1. Install-mode uploads run on a timer, never on a door change
- Install mode lasts 10 minutes. During it the device uploads on a fixed 60-second timer that starts when install mode starts. It never uploads because a door changed.
- Every install-mode upload is padded to the same size step as a normal batch, whether or not anything happened in that minute.
- The LED still blinks on each door change. The light is local, not radio, so it doesn't count as a feed.
- The install check doesn't change. The volunteer does 20 opens, and the key holder decrypts the 60-second uploads on a phone or laptop and confirms 20 events before the volunteer leaves.
- **0003 text, added to the verdict's replacement:** "Install mode, armed once per provisioning (see B4), uploads on a fixed 60-second timer for 10 minutes, padded. Upload timing never depends on door activity."

### B2. Ring frames: one size, nothing readable but a marker
This replaces the `[len][seq][AEAD]` frame in revision-scrappy §3.4 and refines F2.
- **One slot size.** Door events and health rows each use one fixed-size slot, for example 64 B. A dump can't tell a health row from a door event.
- **Slot layout:** `[nonce][AEAD(type ‖ seq ‖ payload ‖ pad)][marker]`.
  - `seq` and the record type sit inside the AEAD.
  - The nonce comes from a keyed function of a per-batch counter, using a subkey of the batch key. It is stored in the clear but looks random, so it doesn't show the order of records within a batch.
  - The 2-byte marker is a fixed constant, written last.
- **Wrapped batch keys are kept out of the ring.** They go to a flash-encrypted store, either NVS or a small key log, together with the slot index where each batch starts. The ring itself never shows where a batch begins, so it can't reveal how many records fell in each 3-hour batch.
- **Finding the write head at boot, without a key.** The ring partition isn't encrypted (F2), so erased space reads as 0xFF.
  - A slot is in use if any of its bytes isn't 0xFF.
  - The head is the first slot that is all 0xFF after the last slot in use.
  - A slot in use with no marker is a torn write. The device logs it as `torn` and skips it. The device detects torn writes on its own, so it doesn't need the commit byte or a key.
- **What a stolen-device dump still shows.** It shows how many slots have been written since the last wrap, which is roughly the total number of records over that span, often weeks. That is a coarser total than the weekly page publishes. It shows no time, no type, no order within a batch, and no batch boundaries.
- **Security gate, added:** a full-ring dump from a unit with at least 7 days of mixed traffic shows nothing but the marker and random-looking bytes. The test passes if (a) every slot is the same length, (b) slot bytes, other than the marker, pass a simple randomness check, and (c) health slots and door slots can't be told apart.
- **Bench item:** check that the key-log or NVS partition is big enough for 90 days of wrapped keys. With 3 holders and 8 batches a day, that is about 200 KB. If it is too small, rotate the batch key every 6 hours instead.

### B3. The 90-day rule on the device uses write times kept per sector
- When a ring sector fills, the device writes that sector's newest-write time, from its own clock, to encrypted NVS. That is one NVS write per 4 KB sector.
- Once the clock has been set since the last cold boot, the device erases tail sectors whose newest-write time is more than 90 days old, oldest first, so the ring stays continuous. Before the clock is set, it expires nothing.
- When it erases a sector, it also deletes the wrapped batch keys used only by that sector. Without its key, any surviving copy of that sector's data can't be read.
- Each expiry is logged in a health row as `expired` with the range of slots erased. The report treats expired data as removed on purpose, not as a gap.
- **Bench test:** backdate one sector's write time by 91 days. The device erases that sector and its keys and logs `expired`. Sectors that aren't backdated are left alone.

### B4. What arms install mode and what uses it up
- **At provisioning:** the unit starts in bench state. Install mode never runs in bench state, so the freezer, power-cut and security gates don't use it up.
- **Arming it:** once the bench gates pass, a key holder removes the cells and runs `latchwing arm-install` over USB.
  - It is a command signed with a holder key, checked against the holder public keys built into the unit.
  - After `dump-sealed`, it is the only other input the USB driver accepts.
  - It sets the NVS flag `install_armed`.
- **Starting it:** on site, inserting the cells powers the unit on. If `install_armed` is set, install mode runs for 10 minutes (B1).
- **Using it up:** the first install window that receives at least one signed ack clears `install_armed`. If no ack arrives, for example because Wi-Fi failed, the flag stays set and the volunteer can retry by reseating the cells, up to 3 windows. After 3 windows the flag clears and the unit runs normally, and a key holder has to bring a hotspot.
- **The verdict's "a magnet in transit" note no longer applies.** Install mode starts on power-on when armed, never on a door change.
- **Gates, added:**
  - "Install mode runs on site, after the bench gates."
  - "After install mode is used up, 5 power cycles never start it again."
  - "An `arm-install` command with an invalid signature is ignored."

## Data and decision text the review found missing

- **D1. 0002, amended to list the health fields.**
  - Health row: `battery_mv` (under load), `boot`, reset reason, `pin_level`, `wakes`, `hours_since_open`, `rssi`, `bounce_rejects`, `chatter`, `missed_edge`, `door_open_long`, `overwritten`, `torn`, `expired`, `low_batt_skip`, `upload_fail` (W1), `temp_c`, `clock_set`, and the latest ack offset.
  - **Wording:** "Health fields that come from door activity (`pin_level`, `wakes`, `hours_since_open`, `door_open_long`, `missed_edge`, `bounce_rejects`, `chatter`) are handled exactly like raw door events: sealed, readable only by named holders, never published, and deleted at 90 days."
  - `clock_reject` is removed (see T2).
- **D2. 0002, names the chip's internal temperature sensor as the only added sensor.**
  - It measures the device, not people.
  - It is read only after the batch key has been drawn, under the same ordering rule as the battery ADC, whether or not it shares the SAR ADC. Confirm in ESP-IDF at the bench.
- **D3. 0003, added:** "The server may email named holders when a device misses uploads. It uses arrival metadata only and never reads sealed contents."
- **D4. 0006, added:** "Report output that leaves a holder's laptop by email carries only the name of the rule that fired and windows of at least 1 hour. It never carries event times or counts."
- **D5. 0004, added:**
  - "The public totals behind the label's QR code are the weekly report, posted by hand by a holder."
  - "They are weekday-by-hour totals for completed weeks only: no current-day data, no date-and-hour cells, and counts under 3 shown as '<3'."
  - "They are never updated automatically."
  - This also settles advisory A1. A status page is still declined.

## Decision-text gaps

- **T1.** The 0002 clock wording becomes "the microcontroller's own RTC, or an external crystal RTC if the bench drift gate fails."
- **T2.** F1 removes `clock_reject` and the 5-minute bound everywhere: the health row, the suspect rules, and the morning check. `synced` means "the clock has been set since the last cold boot." `clock_suspect` (F1) takes over from `clock_reject` in the 0004 gap list.
- **T3.** The 0006 90-day line now covers every pulled copy: "Raw rows, and sealed batches pulled to a holder's machine, are deleted 90 days after they were written, wherever they are."

## Skill-check misses

- **S1.** The install card says "the bottom of the box at least 24 in above the ground, and above where snow piles up." This replaces "low on the wall."
- **S2.** Install step 3 gets an extra check: close the door and run a sheet of paper around the gasket at the contact to confirm it seals. Confirm the box doesn't cover a vent or the condenser grille.
- **S3.** When pin wake comes back on after the chatter lockout, the device reads the pin once. If the pin differs from the last logged state, it logs exactly one change. The log never shows two opens or two closes in a row.
- **S4.** 0006 adds: "If a device goes missing, its identity key is revoked on the Worker the same day."
- **S5.** Before a board is wiped during a holder removal, a holder takes a `dump-sealed` copy or lets the board make its last upload. If neither is possible, the lost slot range is logged as a gap.
- **S6.** The sleep-current gate now includes the time taken to seal and sign, measured on the real C3, in the budget it recalculates.

## Added for spotty Wi-Fi: W1, a cap on upload attempts

The site report is that Wi-Fi at the fridge is spotty, so failed connections need a cap. Detailed connection data from the site visit is still to be confirmed: signal strength in dBm at the mount spot and nearby, how often it drops, and whether there's a sign-in page. W1's timeout and power figures should be checked against that data.
- Each scheduled window gets one connection attempt with an 8-second timeout. If it fails, the device logs `upload_fail` and doesn't retry until the next window. The hotspot fallback stays on the hourly timer only.
- **Cost of the worst case,** with every attempt failing and assuming about 100 mA for 8 s, 8 times a day: about 1.8 mAh/day more. That takes the budget from about 4.5 to about 6.3 mAh/day, and life from about 256 to about 180 days at freezing. This is an estimate, not a measurement, and it is still more than 5 times the trial.
- **Bench item:** measure the current and time of a failed connection on the real board, cold.

## Advisory

- **A2.** If both fallbacks are needed, the PCF8523 plus the polycarbonate box take a unit to about $53–56. If both bench gates call for them, accept going over budget. Cost carries weight 1.
- **A3.** The repo needs the Worker, the `latchwing` CLI including `arm-install`, and the install card, with no secrets, Wi-Fi logins or holder emails in it.
