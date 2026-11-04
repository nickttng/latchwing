# Critique: Team Scrappy, round 01

Reviewing `proposal-hardened.md` and `proposal-pantry.md`, using the review questions in `docs/teams/scrappy.md`. What can be deleted with no loss to the trial? What does it really cost, and how long until it is on a door? Which part is hard to buy, slow to ship, or hard to replace? Is it solving a problem the trial will actually hit?

---

## Proposal: Hardened

### Biggest risk
**The failure the team predicts has no fix in the design, and the fix it does have works too slowly.** Section 10 says the VHB-mounted magnet on the fridge door "will most likely shift or peel within two weeks". It will be applied this winter, on a cold, condensing steel door. The design then finds the problem through `door_open_long` and `missed_edge` rows, which come out only at the weekly `latchwing pull`, and fixing it "takes a site visit". That fails both of the brief's success tests: counts within 5 percent, and no site visit. The trial lasts 30 days, so a magnet that peels in week two and gets spotted in week three ruins about half of the fridge data.

The adhesive rule makes it worse. Step 1 allows installing on any day above 40 °F. 3M's own minimum for the common VHB families (4910, 4950, 5952) is 50 °F, measured on the surface and not on the tape, and 70-100 °F is ideal. So the written steps permit a bond that 3M says not to make.

**Cheapest fix:** (a) Raise the install rule to "surface at 50 °F or warmer", and warm the door and frame themselves, not just the tape. Put a small strip of mechanical retention (a cable tie or clip over the magnet) wherever the door shape allows it. (b) Pull and decrypt daily for the first 3 days, and require a manual tally during that window. Then a shifted magnet costs one visit while the installer is still engaged, not half the trial. (c) Think again about the steel door. The design already adds a spacer because steel shunts the field. Before trusting VHB for the full 30 days, bench-test magnet and spacer on a steel plate in a freezer.

### Findings

**1. The install check in step 4 cannot work as written.**
- *Claim:* "The key holder confirms by phone that the server shows 20 events after the next upload."
- *Failure:* Section 6 says the server holds only ciphertext and "can't read them". It cannot show 20 events. On top of that, uploads run every 2 hours and the box has no lights. In practice the volunteer opens the door 10 times and leaves. Up to 2 hours later, someone has to run a full `latchwing pull` and decrypt before anyone knows the magnet is aligned. If the gap is too wide, the error is found after the volunteer has gone home, and the fix is a second trip. This step also gates the cabinet install ("repeats steps 2-4").
- *Cheapest fix:* Add an install mode like Pantry's: for the first 10 minutes after power-on, upload a batch right after each door change. A key holder on site, or on the phone with a laptop, then runs `latchwing pull` once and reads back "10 opens, 10 closes". If the C3 board is used, add an LED that shows open and closed only during those 10 minutes. That is about $0.05 and one resistor.

**2. Release-mode flash encryption with no update path turns any firmware bug into a new board.**
- *Claim:* Section 6 enables flash encryption in release mode, burns the JTAG eFuses, and sets secure download mode. The design has no OTA.
- *Failure:* ESP-IDF's flash encryption docs say that in release mode the serial bootloader cannot encrypt, and "new plaintext images can ONLY be downloaded using the over-the-air (OTA) scheme." This is first-ever firmware. A debounce bug, a Wi-Fi join that fails on the host's captive-portal guest network, or a padding bug found on day 2 cannot be fixed on the locked units. The cure is new boards, re-provisioned from scratch, and every bench test of the locked build costs a board. The same limit breaks section 6's removal step ("flash a list without their public key") unless it means a new board.
- *What 0006 actually requires:* sealed batches, a hardware RNG, and a debug lock. Records are already AEAD-sealed before they reach flash, so flash encryption protects only the Wi-Fi password and the device identity key. Neither of those exposes door data.
- *Cheapest fix:* Keep Secure Boot V2, the JTAG lock, and secure download mode. Those block serial flash reads and still allow signed reflashing over USB. Drop flash encryption for v1, or keep it in development mode until the bench soak passes. Write down the trade-off: someone who desolders the flash and reads it gets a guest Wi-Fi password and could upload junk batches under that device's identity. Alternatively, keep the full lock but budget two spare boards ($11) and plan on a board swap for any change.

**3. The commit byte does not fit encrypted flash, and it is not needed.**
- *Claim:* Each record ends with "a separate commit byte written last. NOR flash can program the commit byte after the body."
- *Failure:* With flash encryption on, writes go in 16-byte encrypted blocks with 16-byte alignment, so you cannot set a single byte later inside a block that is already written. Either the ring buffer sits in an unencrypted partition (not stated anywhere), or this scheme silently fails. It also isn't needed. Every record already carries a Poly1305 tag. At pull time, a torn record fails tag checking, and the tool can log it as `torn_write` off the device. On the device, finding the next free slot only takes a scan for erased 0xFF space after the last length-prefixed record.
- *Cheapest fix:* Delete the commit byte. Put the buffer in a plain (unencrypted) partition, since its contents are already ciphertext. Do torn-record detection in `latchwing pull`. That is less firmware and less to test.

**4. The PCF8523 is $4.95 and one more breakout to wire, and it accounts for most of the budget overrun.**
- *Claim:* A separate RTC is needed "because the C3's internal RC clock drifts badly in the cold."
- *Failure (or rather, what it costs):* The design syncs time from a signed ack every 2 hours and wakes every hour. ESP-IDF recalibrates the slow clock against the 40 MHz crystal at each wake, so between syncs the drift is bounded by about an hour's worth of RC error. For hourly buckets, a few seconds or even a minute does not matter. The external RTC only earns its place in the case where Wi-Fi is dead for weeks. In that case the team's own assumption says the silence alarm is useless and the trial has already failed "no site visit".
- *Cheapest fix:* Spend one evening putting a bare C3 in a freezer for 48 hours, with and without syncing, and measure the drift. If 2-hour syncs keep it under 1 minute, delete the PCF8523. That brings the device to about $49.60, under budget, with one fewer part to source and solder. The `synced` field in 0002 already makes drift visible.

**5. Twelve uploads a day spend most of the battery and make the most of the cold-sag risk the team names.**
- *Claim:* The design uploads every 2 hours, and the 6-hour silence alarm is presented as a feature. The design admits that "the risk at -20 °F is voltage sag during 350 mA Wi-Fi peaks."
- *Failure:* Uploads are 3.67 of the 6.58 mAh/day, and each one is a sag event. On a -20 °F night that means 12 chances a day for a brownout reboot or a `low_batt_skip`. A 6-hour alarm only helps if someone can respond within 6 hours, and the plan has no such person. The upkeep plan is "a volunteer checks". The heartbeat already shows dead-versus-quiet in the data whatever the upload cadence.
- *Cheapest fix:* Upload every 6 hours, as Pantry does, and alarm at 24 hours. Keep the hourly heartbeat, which needs no radio and costs 0.03 mAh/day. This cuts sag exposures to a third and changes nothing the trial measures.

### Worth stealing
**The hourly heartbeat reads the raw door level and logs `missed_edge` when it disagrees with the last recorded state.** It costs about 0.03 mAh/day and no radio. It turns the most likely counting fault (a shifted magnet, a cut cable, a missed edge) into a row in the data instead of a silent miscount. It also gives a clean, hour-level definition of a gap: no heartbeat in the hour, or a break in the sequence numbers. Every v1 should have this.

### Unchecked claims
- **"19k records … covers 78 days at 250 per day":** the design's own load is 400 events plus 24 heartbeats a day, so 1.75 MB / ~92 B = ~19k records / 424 = **about 45 days**, not 78. It is still more than 30, so the trial is fine.
- **Section 9, "at 5x … the buffer still holds more than 15 days offline":** 19,000 / (2,000 + 24) = **about 9 days**. The claim is wrong. It matters only if Wi-Fi is also down.
- **"VHB adhesive won't bond in the cold … above 40 °F":** checked. 3M lists 50 °F as the minimum surface application temperature for the 4910/4950/5952 families, and 70-100 °F as ideal. 40 °F is too low.
- **Flash encryption release mode and reflashing:** checked against ESP-IDF docs. In release mode, plaintext updates only go through OTA. Encrypted writes are 16-byte blocks. This is the basis for findings 2 and 3.
- **L91 at 3000 mAh:** consistent with Energizer's older datasheet (3000 mAh to 1.0 V at 200 mA). Pantry uses 3500. Neither is load-bearing for 30 days.
- **Hammond 1554C2GY at $18.23 "regional page, unverified for US":** not verified. The clear-lid sibling 1554C2GYCL lists around ¥2,600 (about $17) single-unit at Farnell Japan, so ~$18 is plausible. Shipping is not counted for any part, and that will exceed the $5 overrun on a two-unit order.
- **Deep sleep ~60 µA for the XIAO C3 board:** unmeasured, as stated. It is not load-bearing, because sleep is 1.56 of 6.58 mAh/day.
- **Cloudflare Worker + D1 free tier:** unverified, and low stakes at 24 requests a day.

---

## Proposal: Pantry

### Biggest risk
**Nobody looks at decrypted counts until the trial is over.** The report runs "monthly". The status page and alert email show only check-ins and battery voltage. The install check (5 blinks in install mode) proves the contact switches at minute one, and nothing after. If the contact chatters at partial opening, or the magnet creeps (the team's own "second most likely" failure), or the pairing logic in `latchwing report` drops real visits as ">1 hour opens", the heartbeats keep arriving. The status page stays green, and the first decrypted look on day 30 shows a month of untrustworthy counts with no time left to fix them. The brief's main test is "counts within 5 percent of a manual tally", but the proposal never says when the tally happens or who compares it.

**Cheapest fix:** It costs nothing to build. Write a schedule into the plan: run `latchwing report` on day 1, day 3, day 7, and then weekly. Hold a 2-hour manual tally on day 1 or 2 and compare it against those hour buckets right away. The tool already exists in the design. Only the cadence changes.

### Findings

**1. The rescue-hotspot scan fires on door closes, which sends a "door just closed" radio signal.**
- *Claim:* "If the guest network fails for 24 hours, the device also tries a 'rescue hotspot' network, once per door close and at most every 10 minutes." The 0003 change also says a buffer-threshold trigger "leaks how busy the door is."
- *Failure:* ESP-IDF's default scan is active, so the device sends probe requests. Once the guest Wi-Fi is down, which the team itself names as the most likely first failure, anyone with a $5 Wi-Fi sniffer near the fridge sees a probe burst within seconds of each door close. That amounts to a live "door is open" feed, which the brief forbids as a hard constraint. It is also exactly the timing leak the team removed from upload scheduling. Privacy is pass/fail, so a judge could fail the design on this line alone.
- *Cheapest fix:* Drive the rescue scan from a timer, not from the door: every 15 minutes on a fixed clock, using a passive scan where possible. That is about 96 scans a day at ~1.5 s, roughly 3.6 mAh/day, which is affordable against a 728-day budget. Change step 3 of the minder card to "turn on the hotspot and wait up to 15 minutes."

**2. "Re-provision with the new key list over USB-C" doesn't work after the eFuses this design burns.**
- *Claim:* Section 6 removes a holder by re-provisioning ("a 15-minute visit with a laptop and USB-C"), and `latchwing provision` burns the eFuses for Secure Boot v2, flash encryption, and the JTAG/USB-JTAG lock. The design has no OTA.
- *Failure:* ESP-IDF says that in release-mode flash encryption, "new plaintext images can ONLY be downloaded using the over-the-air (OTA) scheme," and the serial bootloader cannot encrypt. The laptop visit therefore cannot write a new key list or new firmware, unless the app includes a signed config-update command, and none is described. A firmware bug found in week one hits the same wall. The yearly service plan in section 5 assumes the same reflash. For the trial, the risk is a bug that can't be patched, so the only recovery is a board swap and a second bench session.
- *Cheapest fix:* Pick one and say so. (a) Leave flash encryption off for v1, or in development mode. Sealing already protects the events, and Secure Boot plus secure download mode still block reads and allow signed reflashing. (b) Keep the lock and treat the $5.20 board as the unit of re-provisioning: removing a holder means a new provisioned board in the same box. Either way, update the removal and service steps to match.

**3. Step 4 opens the box outdoors on install day.**
- *Claim:* "Open the 4 lid screws, flip the switch on, check the desiccant is blue, and close the lid." Install mode starts at power-on.
- *Failure:* A volunteer in a Chicago winter, with gloves, handles four small screws over snow and reseats an IP66 gasket that has never been checked. Humid outdoor air gets sealed in with a fresh desiccant pack. One pinched gasket or one lost screw undoes the enclosure, and because install mode is tied to power-on, there is no way to switch on at the bench and still get the blink check.
- *Cheapest fix:* Switch on and close the lid at the bench. Trigger install mode with the first door change after power-on, lasting 10 minutes or 20 events, instead of tying it to power-on. The volunteer never opens the box, and step 4 is deleted.

**4. Steel-on-steel will shrink the 3/4" gap, and the stated fallback breaks the budget.**
- *Claim:* The SM-200Q is rated for a 3/4" gap, the install keeps it "under 1/2 inch", and on thick steel the plan is to swap to an SM-226L-3Q for "about $18, which brings the total to about $60."
- *Failure:* The 3/4" figure in Seco-Larm's instructions does not apply to steel mounting. A steel frame and a steel door both shunt the field, which is the condition the team itself expects ("Steel fridge door"). The device may then need the fallback part, which arrives at the site as a $60 device and an order delayed by a week.
- *Cheapest fix:* Do what Hardened does. Add a plastic spacer of about 10 mm under both the contact and the magnet (a $0 scrap block or a stack of VHB layers). Bench-test it on a steel plate before ordering. Keep the SM-226 only as a contingency that has been verified.

**5. The box is held only by VHB on the fridge side, and the install rule allows a cold bond.**
- *Claim:* "Stick the box on the fridge side" and "Warm the adhesive parts in a pocket."
- *Failure:* A pocket warms the tape, not the steel. 3M's minimum for the common VHB families is a 50 °F surface. A box with 3 AA cells at hand height, on a weak bond, is the part most likely to be pulled or to fall, and a fallen box is a 30-day gap. The team names cold VHB as its second-most-likely failure but only fixes it with "a mild day if one comes."
- *Cheapest fix:* The fridge is steel, so mount the box on two rated pot magnets or a magnetic strip ($2-3, already listed as a fallback in section 9) and use VHB only as a secondary hold. That makes winter installs possible and puts no adhesive on the host's fridge. The trade-off is that the box is easier to lift off, so keep the clipped-flat cable, and note that a removed box shows up as a gap.

### Worth stealing
**The 10-minute install mode: the LED blinks on each detected open, the device uploads immediately, and the status page says "checked in just now".** It is the only design here where a volunteer with no key and no laptop can confirm on site, in 20 minutes, that the contact is switching and the device is reaching the server. Once it is done the LED is dark for good, so there is no live signal. It directly scores the brief's "a volunteer can build and install it" criterion, and it fixes Hardened's broken step 4.

A close second is the "decision it changes" line (restock drop time, and cold versus dry shelf space). It is the clearest answer to the brief's "answers a real pantry question."

### Unchecked claims
- **"Rescue hotspot … once per door close":** not a number, but it is the source of finding 1. Check which scan mode the firmware uses (active or passive) before claiming any timing privacy.
- **Flash encryption release mode and USB re-provisioning:** checked against ESP-IDF docs. Plaintext updates only go through OTA. This is the basis for finding 2.
- **SM-200Q "3/4" gap":** checked. Seco-Larm's instructions give 3/4" (19 mm) as the maximum gap. That is not a steel-mounted figure, and Seco-Larm sells a separate steel-door model. Bench-test on steel.
- **Hammond 1554C clear-lid SKU "unverified":** the part exists as 1554C2GYCL (120 x 65 x 40 mm, clear lid, IP68). Farnell Japan lists about ¥2,600 (about $17) for one unit. No US price was found, so expect the top of the $13.04-18.23 range. The total is then about $47, still under $50.
- **"VHB applied below about 10 C bonds weakly":** checked and correct (3M: 50 °F minimum surface temperature for 4910/4950/5952). The fix, warming the tape in a pocket, does not meet it, because the surface must be warm.
- **L91 at 3500 mAh:** unsourced. Energizer's older datasheet gives 3000 mAh to 1.0 V at 200 mA. At 3000, life is about 620 days, which still has a huge margin. Not load-bearing.
- **Deep sleep 15 µA board level, "vendor forum figure":** unmeasured, as stated. Even at 10x, the team's own math still gives about 300 days. Not load-bearing.
- **XIAO ESP32C6 at $5.20 "10+ tier":** the trial buys 2. Single-unit price and shipping are not stated. The difference is small.
- **Cloudflare Worker + R2 free tier, R2 lifecycle deletion at 90 days:** unverified, and low stakes.

### Sources checked
- [ESP-IDF flash encryption, ESP32-C3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/security/flash-encryption.html)
- [3M VHB application temperatures (Strouse summary of 3M guidance)](https://www.strouse.com/blog/these-are-your-application-temperatures-for-3m-vhb-tape)
- [Energizer L91 datasheet (Farnell copy)](https://www.farnell.com/datasheets/9583.pdf)
- [Seco-Larm SM-200 instructions](https://cdn.shopify.com/s/files/1/0269/1088/3897/files/SM-200W.pdf)
- [Hammond 1554C2GYCL at Farnell Japan](https://jp.farnell.com/en-JP/hammond/1554c2gycl/enclosure-watertight-pc/dp/1877046)
