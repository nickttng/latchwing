# Critique from Team Hardened, round 01

Reviewed against the Hardened charter: assume the worst week in January, with nobody coming to help. Both proposals are close cousins: XIAO ESP32 board, wired reed contact, 3x L91 into the 5 V pin, 6-hour heartbeat and upload, Cloudflare Worker, one device per door. So several findings apply to both. Each one is written out in full under each proposal, so either section can be read alone.

Sources checked for this critique (2026-11-03):
- Seeed XIAO ESP32C3 wiki: deep sleep is 44 µA (the features list says about 43 µA). Using 5 V as an input needs "some sort of diode (schottky, signal, power)" in series. Deep-sleep wake works on D0 to D3.
- Seeed XIAO ESP32C6 wiki: deep sleep is 15 µA typical at a 3.8 V supply. It says nothing about feeding the 5 V pin or about a diode.
- 3M VHB 5952 data sheet: minimum application temperature is 10 °C (50 °F), with 21 to 38 °C as the ideal range.
- Energizer L91 data sheet: rated -40 °C to 60 °C. Its cold capacity data is graphed and only partly published as numbers, and the main cold test point is 0 °C.

---

## Proposal: Team Scrappy

### Biggest risk
**Mounting the sensor in winter.** The plan sticks an unverified MC-38 clone onto a steel fridge with VHB in Chicago winter, and the firmware does nothing to limit the chatter that follows. Scrappy names misalignment as its own first failure. Its only fix is a volunteer site visit, which breaks the trial's success rule. 3M's minimum application temperature for VHB 5952 is 50 °F, and a hand warmer held on a steel panel at -20 °F for 2 minutes won't keep the metal above that while the tape wets out. Once the gap drifts, the counts go wrong and the battery and buffer drain (finding 2). The weekly report shows it after the damage is done.

### Findings

**1. The sensor mount and gap are unproven where it counts.**
- *Claim:* VHB plus a 2-minute hand warmer gives a good bond. The gap stays under 10 mm, checked on a steel offcut first.
- *Failure:* The install happens on a January day at 10 °F. The steel's own thermal mass pulls it straight back below 50 °F, so the VHB never wets out. Each slam shears the magnet a little. By week two the gap passes the MC-38's real switching distance on steel, which is unknown for a no-name clone. The door now misses closes, so opens go unpaired and get dropped from the counts. The 5 percent accuracy target is gone, and getting it back needs a site visit.
- *Cheapest fix:*
  - Buy a contact with a published gap figure on steel (a branded wide-gap part), and bench-test it on the offcut at freezer temperature.
  - Make a substrate temperature of at least 50 °F a pass/fail install step. A heat gun plus a $10 IR thermometer can do it, or install on the first mild day.
  - On the wood cabinet, screw both parts down rather than taping them.

**2. Nothing contains chatter, and overwrites are silent.**
- *Claim:* A 50 ms settle and 500 ms lockout give clean events. A 50 percent-full upload trigger guards against overflow. The buffer overwrites its oldest records when full (per 0003).
- *Failure:* A loose magnet sits right at the switching edge, and the fridge compressor's vibration makes it toggle. Level wake plus the 500 ms lockout allows about 2 events/s, which is about 170,000 wakes a day. The power budget's own numbers give 170,000 x 0.25 s x 25 mA ≈ 300 mAh/day, so the derated 1,150 mAh (3,000 x 0.5 / 1.3) lasts about 4 days. If Wi-Fi is also down, 2 events/s x 90 B fills the 1.5 MB ring in about 2.3 hours. The ring then overwrites good, unsent records, and nothing in the data says so. That breaks our principle that any failure must show in the data.
- *Cheapest fix (firmware only):*
  - If there are more than N wakes in M minutes (say 20 in 1 minute), write a `chatter` health row. Then disable pin wake for 10 minutes and mark that window as a gap.
  - Add an `overwritten` count to the health row.
  - Never let the overflow guard drop unacked records without logging it.

**3. A sensor stuck closed reads as a quiet door.**
- *Claim:* Health rows every 6 hours tell a dead device from a quiet one. A torn wire reads as stuck open and gets flagged.
- *Failure:* Pinched cable insulation, condensation bridging the screw terminals, or a reed welded shut in the cold all hold the pin at "closed" for good. The MCU keeps writing healthy heartbeats, so every hour bucket counts as data, and the report shows true zeros at a 24/7 fridge. Heartbeats prove the chip is alive, not the sensor. That is exactly the dead-versus-quiet question in the brief, and 0004's gap rule is broken without anyone noticing.
- *Cheapest fix:*
  - Put the raw pin level at heartbeat time and a wake count since the last heartbeat in the health row. It is sealed like everything else.
  - Have `latchwing report` flag any fridge window with no opens for longer than a set limit (start at 12 hours) as "sensor suspect", shown as a gap, not zero.
  - Check the flag in the weekly run.

**4. A power cut mid-write is not specified.**
- *Claim:* Sealed JSON-lines records go into a 1.5 MB flash ring. The read pointer advances on a valid ack.
- *Failure:* At -20 °F, an upload's TX peak (the ESP32-C3 draws a few hundred mA at full TX power, not the 110 mA average in the budget) pulls three cold L91s down. After the 1N5817 drop (which grows in the cold) and the LDO dropout, the chip browns out in the middle of a flash append or a sector erase at the ring's wrap point. On reboot, the proposal doesn't say how the device finds the write head, where the read pointer is saved, or what happens to a half-written record. A torn sector can make the device skip or re-upload records, or stop the report tool partway through.
- *Cheapest fix:*
  - Frame each record as length + sealed record. The AEAD tag already serves as a free commit check.
  - On boot, scan for the last valid record and write a `torn` count in the health row.
  - Save the read pointer in NVS, which is wear-levelled and handles power loss.
  - Add one freezer test: cut power during an upload while the cells are at -20 °C.

**5. A yank on the box loads the sensor.**
- *Claim:* The box sits on magnets, so a yank pops it off harmlessly. A torn wire reads as stuck open.
- *Failure:* A box that pops off hangs by its cable from the sensor on the frame. The cable's weight and any further tugging peel the cold-applied sensor, or pull its leads out of the terminals. Rubber-coated cup magnets also lose shear grip in the cold, so they can slide down the fridge wall under constant door-slam vibration. That keeps pulling the cable. Either way a tug turns into a count failure, not a clean gap.
- *Cheapest fix:*
  - Add a strain-relief cable clip (VHB-backed or screwed) within 5 cm of the sensor, so a pull on the box stops at the clip.
  - Use a short drip loop, and keep the box low enough that it falls only a little if it pops off.

### Worth stealing
**Saying where the "RAM only" batch key lives across deep sleep.** It goes in RTC slow memory, is drawn right after an upload while the RF entropy source is on, is lost on power-off, and a cold boot starts a new batch. This settles a real gap in 0006: on these chips, deep sleep wipes normal RAM. Nobody else is forced to answer this until firmware time, when a careless choice either reseals on every wake or keeps the key in flash. A close second: `bounce_rejects` in the health row, which is the start of a sensor-health signal.

### Unchecked claims
- **Deep sleep 44 µA: checked.** It is on the Seeed wiki. But that figure is not measured through the 5 V pin, the Schottky, and the onboard LDO, which is the path this design uses. The 20 µA regulator figure is a guess, and the 10x-worse case (≈90 days, arithmetic checked) is the real safety net. Measure it in the freezer.
- **The 5 V pin needs a diode: checked.** The wiki says so, and Scrappy includes one.
- **GPIO3 (D1) can wake from deep sleep: checked.** Wake is supported on D0 to D3.
- **Hand warmer is enough for VHB: contradicted.** The 3M minimum is the substrate at 50 °F (finding 1).
- **Cold derating of 0.5:** this comes from the skill, not the data sheet. L91 is rated to -40 °C, and at low drain 0.5 is probably conservative. The pulse behaviour at -29 °C under TX peaks is unchecked, and that is the number that matters (finding 4).
- **355 days:** the arithmetic is correct (1,500 / 3.25 / 1.3). It covers normal behaviour only. Chatter takes it to about 4 days (finding 2).
- **Upload of 110 mA for 10 s with TLS and signing, cold:** unverified. Radio time grows with weak RSSI and retries.
- **Internal RC clock drift:** not quantified. ESP-IDF warns that the internal RC slow clock drifts with temperature. Over 6 hours in a box that swings between night cold and day sun, the drift could pass 5 minutes routinely. The "flag past 5 minutes" rule would then flag almost every batch and stop meaning anything. Measure drift in the freezer test before trusting the flag.
- **MC-38 gap on steel, cup magnet holding force, Worker and R2 free-tier limits:** all unverified. Only the gap affects the trial.

---

## Proposal: Team Pantry

### Biggest risk
**Nobody looks at the counts until the month is over.** The only alert during the trial is "the device missed check-ins". That proves the radio works, not the sensor. A contact that peels off the cold-applied VHB (Pantry's own second failure), chatters, or sticks closed on day 3 produces 27 days of healthy heartbeats with wrong or zero counts. Those are found at the monthly `latchwing report`, which falls on day 30, the end of the trial. The design is good at not losing data and has no way to find out that the data is bad while there is still time to fix it.

### Findings

**1. Liveness covers the MCU only, and the review cadence is monthly.**
- *Claim:* A sealed heartbeat every 6 hours plus a 24-hour missed-check-in email tells dead from quiet. A heartbeat interval with no events is a true zero.
- *Failure:*
  - Condensation bridges the contact terminals, or the reed welds shut at -20 °F, or a pinched cable shorts. The pin then reads "closed" for good. Heartbeats keep arriving, the status page stays green, the minder gets no email, and the monthly chart shows the fridge as empty. That is a false zero, which 0004 forbids.
  - A peeled contact that half-works gives drifting counts that are just as invisible.
- *Cheapest fix:*
  - Run `latchwing report` weekly during the trial, not monthly.
  - Add a wake count and the raw pin level to each health row (sealed).
  - Make the report flag any fridge window with no opens for longer than a set limit (start at 12 hours) as "sensor suspect", shown hatched like a gap.

**2. Rescue-hotspot mode is not in the power budget, and it ties the radio to door activity.**
- *Claim:* After 24 hours without the guest network, the device tries the rescue hotspot "once per door close and at most every 10 minutes." The upload schedule is fixed, so upload timing says nothing about traffic.
- *Failure:*
  - The host changes the Wi-Fi password on day 5, which is Pantry's own predicted first failure.
  - From then on, the device makes up to 144 failed 15 s attempts a day at 100 mA, about 60 mAh/day. Against a derated 1,346 mAh (3,500 x 0.5 / 1.3), that is about 22 days. At the assumed 50 visits/day it is closer to 20 mAh/day, still roughly 10x the whole normal budget.
  - At -20 °F, with the minder slow to respond, the batteries can run out before the rescue hotspot ever shows up. The heartbeats saved in flash survive, but the end of the trial becomes a real gap.
  - Radio activity now follows door closes, so the RF timing leaks traffic that the padded fixed schedule was designed to hide.
- *Cheapest fix:*
  - Try the rescue network on a fixed timer, not on door events. For example, once an hour, and only in the 2 hours after each missed 6-hour slot.
  - Add that line to the power table and say how many days rescue mode can last.

**3. There is no chatter limit, and the overflow guard is gone.**
- *Claim:* A threshold trigger leaks busyness and "the buffer cannot fill in 6 hours," so 0003's threshold upload is removed.
- *Failure:*
  - A magnet that has crept to the edge of the switching gap toggles under compressor vibration. With a 500 ms lockout that is up to 2 events/s, so 2 x 80 B fills the ~1 MB buffer in about 1.7 hours.
  - Under 0003 the ring then overwrites the oldest records, which are good unsent data, silently.
  - The same storm costs about 170,000 x 0.15 s x 25 mA ≈ 180 mAh/day, which empties the derated cells in about a week.
  - "Cannot fill in 6 hours" holds only for a healthy sensor. If the Wi-Fi is also down (failure 1 again), the buffer has no way to drain at all.
- *Cheapest fix:*
  - Firmware rate limit: if there are more than N wakes in M minutes, write a `chatter` health row, disable pin wake for 10 minutes, and mark a gap.
  - Add an `overwritten` count to the health row.
  - Overflow without a record breaks our principle 2, and a log line costs nothing.

**4. The 5 V pin is fed with no diode, then provisioned over USB-C.**
- *Claim:* Three L91 cells feed the board's 5 V pin directly. Provisioning, and the 15-minute re-provisioning visit, run over USB-C. The device "ships with batteries in."
- *Failure:*
  - On the XIAO ESP32C3, Seeed's wiki says an external source on the 5 V pin needs a series diode, because that pin is tied to USB VBUS. The C6 wiki doesn't say either way, so this must be checked on the schematic.
  - If the C6 is built the same way, the next re-provisioning visit with the battery switch left on puts laptop USB 5 V in parallel with three part-used lithium primaries at ~4.6 V. That pushes charge current into cells that must never be charged.
  - A fresh pack at 5.4 V instead pushes back into the laptop port.
  - Separately, the holder's slide switch is now the only series contact for the whole device, sitting in a box that a volunteer opens in winter air. An oxidised, intermittent switch causes brownout resets in the middle of a write.
- *Cheapest fix:*
  - Add a $0.10 1N5817 Schottky in series, as Scrappy does, and put its drop in the budget.
  - Add "switch off before USB" as a printed step on the bench card.
  - Use a holder whose switch is rated for the cold, or wire the cells straight to the terminal block and rely on the lid screws.

**5. The batch key across deep sleep and a power cut mid-write are not specified.**
- *Claim:* "A new batch starts right after each upload, while the radio is on, so the hardware RNG has true entropy." Records are kept in a ~1 MB flash buffer.
- *Failure:*
  - On the ESP32-C6, deep sleep wipes main SRAM. If the batch key lives there (the obvious reading of "RAM only"), every wake from a door event is a cold start, so the device starts a new batch per event. Each one draws from the RNG with the radio off, which breaks the entropy claim. Each one also writes a fresh `crypto_box_seal` header for every holder to flash, and spends extra power on wake.
  - The proposal also doesn't say how a write cut off by a volunteer flipping the switch, swapping batteries, or a cold TX brownout gets detected. The device needs to find the write head again, and the report needs to count torn records instead of stopping.
- *Cheapest fix:*
  - Keep the data key and nonce counter in LP/RTC memory, which survives deep sleep and is wiped on power-off (Scrappy's answer). Start a new batch on cold boot only.
  - Frame each record as length + AEAD record, scan for the last valid record on boot, and write a `torn` count in the health row.
  - Add one freezer test that cuts power during an upload.

### Worth stealing
**The paper backup for each key holder, plus per-holder server tokens.** Under 0006, losing every private key loses all the data, and with only 2 or 3 volunteers holding keys on laptops that is a likely way for a trial to end up with nothing. A passphrase-protected key file with a sealed paper copy held by the coordinator closes the most likely silent-total-loss path at almost no cost. It also gives same-day removal through tokens without waiting for re-provisioning. Every team should adopt it. A close second: the clear lid, so anyone can check the desiccant colour without opening the box.

### Unchecked claims
- **Deep sleep 15 µA, labelled a "vendor forum figure": partly checked.** It is on the Seeed C6 wiki, at a 3.8 V supply, which probably means the battery pad. It is not measured through the 5 V pin and LDO this design uses, and the power table has no regulator or leakage line, unlike Scrappy's.
- **"If sleep is 10x higher, life is still about 300 days":** the arithmetic is off. 0.15 mA x 24 = 3.6 mAh, plus the other rows makes 5.09 mAh/day, and 1,750 / 5.09 / 1.3 ≈ **264 days**. That doesn't matter for the trial, but it should be fixed.
- **728 days:** the arithmetic is correct. It leaves out rescue mode (finding 2) and chatter (finding 3), which are the states most likely to happen in January.
- **"Coldest week reaches -20 C; L91 rated to -40 C":** the L91 rating is checked. But the charter's design point is -20 °F (-29 °C). -20 °C is only -4 °F, so the cold assumption is about 25 °F too warm. L91 still covers -29 °C at low drain. Pulse performance under TX peaks at -29 °C is unchecked.
- **Battery swap at 3.9 V:** unvalidated. L91 has a flat discharge curve, so voltage gives little warning, and cold sag will trip a 3.9 V threshold early on the coldest nights. Log the voltage under load (during TX), not at rest, and treat this threshold as a guess.
- **"Board is coated so opening the lid is harmless":** coating protects the board, not the battery contacts, the switch, or the screw terminals. Those are the parts condensation will reach.
- **"Warm the adhesive parts in a pocket":** this warms the tape, not the steel. 3M's 50 °F minimum applies to the surface.
- **SM-200Q 3/4" gap:** that is the vendor figure on non-ferrous mounting. On steel it is unknown, and the fallback part breaks the budget ($60).
- **Status page shows battery voltage:** this contradicts "the server keeps only device ID, batch ID, and check-in time." Battery voltage is not people data, so the fix is a sentence: say it is sent unsealed.
- **Hammond 1554C clear-lid SKU, Worker and R2 free tier:** unverified, and low impact.
