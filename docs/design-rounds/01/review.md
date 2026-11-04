# Review, round 01

Reviewer: design-review. Scope: the pick in `verdict.md`, which is `revision-scrappy.md` plus grafts 1–8 and fixes F1–F4. I checked it against the README principles, decisions 0001–0006, the brief's hard constraints, and the skills in `skills/`. Date: 2026-11-03.

## Result: needs changes

None of the hard constraints is changed. Nothing adds cameras, audio, presence sensing or identification. Nothing publishes raw rows. Only named holders' keys open the sealed data. Two items below (B1 and B2) come close to the "no live feed" and "stolen device" constraints as written, so they must be fixed before the bench gates. Every departure the verdict lists has replacement wording in section 6, except the ones flagged here.

## Techniques touched

All six: `event-driven-wake` (0001), `append-only-events` (0002), `store-and-forward` (0003), `time-bucketing` (0004), `enclosure-and-power` (0005), `access-control` (0006).

## Blocking: fix before bench gates

- **B1. Install-mode uploads have no decision change, and they contradict the new 0003 text.** In install mode, "each batch uploads right away" on every door change. The proposed 0003 wording says the 3-hour schedule plus the 50-percent guard is "the only other trigger." So for 10 minutes the RF timing shows each door change, which is the live door signal 0004 rejects. Graft 1 limits how often this happens but does not cover it with a decision. Fix: in install mode, upload on a fixed 60 s timer, padded, not on each door change. Then add to the 0003 replacement: "Install mode, armed once per provisioning, uploads on a fixed 60-second timer for 10 minutes."
- **B2. F2 exposes plaintext frame headers on a stolen device.** F2 moves the ring out of flash encryption. The frame is `[len][seq][AEAD]`, so a flash dump shows `len` and `seq` in the clear. Health rows are a different length from door events and are written every hour, so a dump shows door counts per hour and their order. That fails the `access-control` check "a flash dump shows no readable events," and it contradicts the 0002 text that puts `seq` "in the sealed envelope." Fix: pad every frame, door and health alike, to one fixed size. Move `seq` inside the AEAD, or derive it from the nonce counter. Leave only a fixed marker in the clear for the boot scan. Add a plaintext-header check to the security gate.
- **B3. F4 can't be carried out as written.** The device can't read record ages from sealed frames, and after a power cut it no longer has the key. Fix: keep a "newest write time" for each sector in encrypted NVS, not in plaintext in the ring. Base it on the corrected clock, and expire sectors by it. Add a test: a sector backdated past 90 days gets erased.
- **B4. Graft 1 conflicts with the bench gates on trial units.** Install mode is armed "once per provisioning." The trial units are provisioned before their freezer, power-cut and security gates, so install mode would run on the bench and be gone by install day. Step 5 of the install card (20 blinks, then 3 long blinks) would then fail. The verdict's note "a magnet in transit could start install mode early" also suggests a different trigger than power-on. Fix: say exactly what arms install mode and what uses it up. For example: armed at provisioning, used up only by the first signed ack received after a holder sets an "install" flag. Then add "install mode runs on site after the bench gates" as a gate.

## New fields, sensors and data flows not covered by a decision change

- **D1. Health-row fields go past 0002's "battery, reboots."** The pick adds `pin_level`, `wakes`, `hours_since_open`, `rssi`, `bounce_rejects`, `torn`, `low_batt_skip`, `missed_edge`, `door_open_long`, `temp_c` and reset reason. The `append-only-events` skill says any new field needs a decision. `pin_level`, `wakes`, `hours_since_open`, `door_open_long`, `missed_edge` and `bounce_rejects` describe door activity, so they don't "say nothing about people." Fix: amend 0002 to list the health fields. State that the door-derived ones are handled exactly like raw events: sealed, never published, deleted at 90 days.
- **D2. `temp_c` is a new sensor with no decision text.** Fix: name the internal temperature sensor in the 0002 amendment. Check in ESP-IDF whether the C3 temperature sensor shares the SAR ADC. If it does, read it only after the batch key has been drawn, under the same ordering rule as the battery ADC.
- **D3. The Worker's 12-hour silence email is a new way to move data.** It uses server-side upload metadata, and nothing in section 6 covers it. Fix: add to 0003 or 0006: "The server may email named holders when a device misses uploads, using arrival metadata only."
- **D4. The morning OK-or-rule email takes report output off the laptop through an email provider.** It is not covered. Fix: add to 0006 that the email carries only the rule name and windows of at least 1 hour, never event times.
- **D5. Graft 8's QR "totals" page is a public channel with no rule against it becoming live.** It also sits uneasily with the verdict's "Status page: declined." Fix: say the totals are the weekly report, posted by a holder, with buckets of at least 1 hour and no current-day data, and never updated automatically.

## Decision-text gaps

- **T1. The 0002 replacement rules out the declared clock fallback.** "The clock is the microcontroller's own RTC" leaves no room for the PCF8523 if the drift gate fails. Fix: add "or an external crystal RTC, if the bench drift gate fails."
- **T2. F1 leaves `clock_reject` and the 5-minute bound in the picked design.** The revision's health row and suspect rules still have them, and `synced` now has no definition. Fix: say F1 removes `clock_reject` and the 5-minute bound. Define `synced` as "the clock has been set since the last cold boot."
- **T3. Pulled sealed batches on holder laptops aren't covered by the 90-day rule.** The revision covers R2 and "local raw copies," but not the sealed blobs a holder has pulled. Holders can still open those. Fix: extend 0006's 90-day line to every pulled sealed copy.

## Skill-check misses

- **S1. `enclosure-and-power`: "low on the wall" conflicts with "mounted clear of water spray and snow pile-up."** Fix: set a minimum height above ground on the install card.
- **S2. `enclosure-and-power`: the install card has no check that door seals and vents are clear.** The contact sits at the latch edge, next to the gasket. Fix: add a step to step 3: shut the door and check that the gasket seals all the way round, and that the box doesn't cover a vent.
- **S3. `event-driven-wake`: no rule for what happens when pin wake comes back after the chatter lockout.** Fix: when wake comes back on, re-read the pin and emit one state change if it differs, so the log never shows two opens or two closes in a row.
- **S4. `access-control`: no rule for revoking a missing device.** Fix: add "revoke a missing device's identity key on the Worker the same day."
- **S5. `access-control` and holder removal: an old board's unsent ring is wiped.** Fix: let the old board upload its last batch, or take a `dump-sealed` copy, before wiping. Otherwise log the lost range as a gap.
- **S6. `enclosure-and-power`: the time spent sealing and signing is assumed, not measured.** Fix: add measured crypto time, on the real C3, to the sleep-current gate's budget recalculation.

## Advisory

- **A1. 0004 says individual visits can't be picked out, but an hourly count of 1 on a quiet cabinet can still show one visit.** Fix: publish weekday-by-hour totals for the week, not counts per date and hour, or show counts under 3 as "<3."
- **A2. Cost: if both fallbacks are needed, the per-device cost goes past $50.** The PCF8523 plus the polycarbonate box comes to about $53–56. Fix: note this in section 7, or accept going over budget if both are triggered.
- **A3. The repo needs to hold the Worker, the `latchwing` CLI, and the install card, for "Open and documented."** No secrets, Wi-Fi logins or holder emails go in the repo.

## Covered departures, checked and fine

- 0001: the hourly timer wake and the chatter lockout.
- 0002: setting the clock once after each cold boot, interpolation, and `seq` in the envelope (but see B2).
- 0003: acks delete nothing, the fixed schedule, and the hotspot and USB-sealed transports (but see B1).
- 0004: the wider gap definition.
- 0005: unchanged.
- 0006: the key in RTC memory, removal within 7 days, the account owners, and the F3 keychain rule.
