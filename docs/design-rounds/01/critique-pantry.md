# Critique from Team Pantry, round 01

Reviewer for Team Pantry. Lens: the volunteer who installs it, the key holder who reads it, and the neighbor who sees it. Both proposals pass on privacy and data access as written, with one gap in Scrappy (finding S2) that has to be closed to keep that pass.

---

## Proposal: Scrappy

### Biggest risk
The volunteer can't confirm at the door that the install worked. Step 5 of the install card ("count 20 blinks... three long blinks means the first upload worked") needs an LED the firmware can control. The Seeed XIAO ESP32-C3 has no user LED. Seeed's wiki says "there is no LED_BUILTIN available for the XIAO ESP32C3", and the only LED on the board is the charge LED. The BOM adds no LED. So the volunteer closes the lid with no idea whether the sensor is aligned or the Wi-Fi joined. The proposal names misalignment as its own first failure, and the first sign of it is the 24-hour email or the weekly report. By then the volunteer is gone, and fixing it takes the site visit the trial is supposed to avoid.

### Findings

**S1. The install check depends on a part that doesn't exist.**
- *Claim:* The LED blinks once per door change for 10 minutes, and three long blinks confirm the first upload.
- *Failure:* A volunteer at the fridge in January opens and closes the door 10 times and sees nothing, because there is no LED. They either give up and call the key holder, or close the lid and leave. If the magnet is 12 mm off, nobody finds out until the first weekly report shows chatter or no events.
- *Cheapest fix:* Add a 3 mm LED and a 330 ohm resistor on a spare pin (D10, as in Seeed's own Blink example), for about $0.10. Keep it in install mode only (first 10 minutes after the batteries go in), so the "LED off after install" promise still holds. Make the first upload happen at boot, not at the first 6-hour timer, so the three long blinks can actually appear while the volunteer is there.

**S2. A cold boot draws a batch key without a proven entropy source.**
- *Claim:* The data key is drawn right after an upload while Wi-Fi is on, because the C3's RNG is only fully random with the RF or bootloader entropy source enabled. "A cold boot starts a new batch."
- *Failure:* The first batch of the trial starts at a cold boot (batteries inserted), and so does every brownout reset afterward. No upload has happened yet, so the RF is off and the proposal names no other entropy source. A guessable data key means a copy of that batch taken from R2 or the network can be opened without a holder's private key. Privacy is pass or fail, so this one sentence decides the gate.
- *Cheapest fix:* At every batch start, including cold boot, wrap key generation in `bootloader_random_enable()` / `bootloader_random_disable()`, which is what Hardened specifies. This is one line, plus a test that two cold boots in a row produce different wrapped keys.

**S3. The Wi-Fi has no fallback, so the silence email reports a problem nobody can fix.**
- *Claim:* Guest Wi-Fi credentials go into encrypted NVS. There is no OTA. If Wi-Fi fails, a key holder does one USB pull at the end.
- *Failure:* In week 1 the host business changes its guest password, or the guest network turns out to have a "click to accept" captive portal that a headless device can't get past. The 24-hour email fires. The key holder can't change the credentials, because flash encryption, secure download mode, and no OTA lock them in. The only remedy is swapping in a newly provisioned board, which is a site visit. The volunteer's step 1 checks for "two bars" and nothing else. The pantry charter asks who chases the network when it fails. Here it is the key holder, with no tool for it.
- *Cheapest fix:* Provision a second SSID, the key holder's phone hotspot, as Hardened does. Change step 1 to: "Join the guest Wi-Fi on your phone with the password and open any web page. If a sign-in screen appears, stop and tell the key holder." Also ask the host to tell the pantry before changing the password.

**S4. A magnet-mounted box gets pulled off, and the pull tears out the sensor.**
- *Claim:* The box is held on by magnets, "so a yank pops it off without damaging the fridge."
- *Failure:* The box is wired to the reed contact taped to the door frame. Someone curious or annoyed pulls the box off. It swings on the sensor lead, peels the VHB sensor off the frame, or gets carried away or set back down somewhere else. Every outcome after that is a gap or a false count until a visit. A box that comes off in one tug also tells passers-by that it is meant to be handled, which works against "looks harmless."
- *Cheapest fix:* Put strain relief on the sensor lead (a cable clip or zip-tie anchor on the frame, about 5 cm from the sensor) so a pull loads the gland and not the tape. On the cabinet, use the screws that are already planned. On the fridge, add a short steel lanyard or strap so a popped box hangs instead of leaving.

**S5. A stuck-closed sensor is reported as a quiet door.**
- *Claim:* An hour counts as data if health rows exist within 6 hours on both sides. A torn wire reads as stuck open, which gets flagged.
- *Failure:* On a steel fridge, a door magnet whose tape lets go won't fall to the ground. It snaps onto the nearest steel, which is often the frame right next to the reed. The reed then reads "closed" forever, health rows keep arriving, and every hour is reported as a real zero. A key holder reading "zero opens Tuesday afternoon" might move a delivery. The "open over 1 hour" check never fires, and the health row doesn't record the raw door level.
- *Cheapest fix:* Add a report rule for the fridge door only: a run of 4 or more hours with no events in daytime (06:00 to 22:00) is marked "suspect" and shown as a gap. Tune the threshold from the first week's data. The cabinet keeps the current rule. Add a raw pin level to the health row, as Hardened does.

### Worth stealing
**The key holder's at-home pre-test: 24 hours in a freezer, then 20 tallied opens, before anything reaches the volunteer.** It finds dead boards, cold brownouts, and firmware bugs on a kitchen counter instead of at a public fridge. It costs one evening. It also means the volunteer's job is only the mechanical install, which is the part only they can do. Every proposal should require it.

### Unchecked claims
- **Deep sleep 44 µA:** checked. The Seeed wiki lists 44 µA, and "about 43 µA" elsewhere. That figure is for the battery pads. Scrappy powers through the 5V pin, so the regulator and charger path adds an unmeasured draw. They flag this and leave a 10x margin. It's fine for 30 days.
- **5V pin needs a diode:** checked. The wiki requires a diode between the external source and the 5V pin. The 1N5817 meets that.
- **No 32 kHz crystal:** consistent with the wiki, which mentions none. The internal RC drifts with temperature, and a fridge's side wall swings day to night. The "flag batches past 5 minutes" rule may end up flagging most batches. Measure drift over 6 hours during the 24-hour freezer test, and correct by linear interpolation between ack syncs before trusting hourly buckets.
- **"RTC slow memory":** the ESP32-C3 has only RTC fast memory (8 KB) that survives deep sleep. `RTC_DATA_ATTR` still works, so the design holds, but the provisioning doc should name it correctly.
- **Cold voltage under Wi-Fi peaks:** not addressed. Three L91s in the cold, then a Schottky drop of about 0.4 V at a 350 mA peak, then the LDO dropout, may brown out during uploads. There's no bulk capacitor and no low-battery skip. Copy Hardened's 470 µF polymer cap and `low_batt_skip`.
- **The Worker emails key holders on free tier:** unverified, and it matters because it is the only liveness alarm. On the free plan, Cloudflare's send-email binding delivers only to verified destination addresses. Confirm it works before the trial, or use another free mailer.
- **MC-38 "normally open":** alarm-contact sellers use "NO" and "NC" in opposite senses. Have firmware learn the closed-door level at install rather than hard-coding it.
- **Total ~$36:** arithmetic checks ($35.98). Most line items are marked unverified, but the margin under $50 absorbs them.

---

## Proposal: Hardened

### Biggest risk
The proposal predicts its own most likely failure and doesn't fix it. Section 10 says the VHB-mounted magnet on the cold, condensing steel door "will most likely shift or peel within two weeks," and that repair takes a site visit. The trial requires 30 days with no visit, and the install happens in a Chicago winter. A design whose authors expect the critical joint to fail by day 14 has spent its effort on the clock, the commit byte, and the IP68 box, and left the one part that slams hundreds of times a day held on by tape.

### Findings

**H1. A magnet that comes loose reads as "real zero", not as a gap.**
- *Claim:* "An hour with a heartbeat and no events is a real zero." Section 10 says a shifted magnet shows up as `door_open_long` or `missed_edge`.
- *Failure:* This is the proposal's own first failure, on a steel door. A neodymium actuator that peels off doesn't drop to the ground. It snaps to the nearest steel, which is the fridge cabinet beside the reed, and holds the reed closed. The pin then reads "closed" every hour, so `missed_edge` (raw level equals last state) and `door_open_long` both stay silent. The data reports clean zeros with valid heartbeats for the rest of the trial. That is a dead sensor shown as a quiet door, the failure 0004 forbids. It also misleads the pantry exactly when it matters: "Nobody used the fridge Thursday, skip the restock."
- *Cheapest fix:* A report rule for the fridge door only: 4 or more daytime hours with heartbeats but no events are marked "suspect" and shown as a gap. Tune the threshold in week 1. The server can raise the same alarm from the padded-batch metadata only if padding still reveals "zero events", so do this in the report, not on the server.

**H2. The cold-install guidance is wrong, and the moving part depends on tape.**
- *Claim:* VHB is fine above 40 °F, or below that after a hand warmer is held on the surface for 10 minutes.
- *Failure:* 3M doesn't recommend applying VHB to surfaces below 50 °F. A hand warmer on a steel fridge door loses its heat into the steel as fast as it supplies it. With the trial "starting this winter," almost every install day is below 40 °F. The volunteer follows the sheet exactly and gets a joint that peels.
- *Cheapest fix:* Don't rely on tape on the moving side. Fix the actuator magnet mechanically, for example with a zip-tie or hose-clamp holder on the door handle, or a strap around the door edge where the gasket allows. Use VHB only on the stationary side, near the 1 m reed cable. Change the threshold to 50 °F, and add a step: "tug the magnet firmly; if it moves, stop."

**H3. The install check can't work as written.**
- *Claim:* "The key holder confirms by phone that the server shows 20 events after the next upload, or asks the volunteer to wait for the hourly heartbeat."
- *Failure:* The server can't read events. Batches are sealed to the key holders, and they are padded to fixed size steps specifically so their size doesn't reveal the count. The server can't "show 20 events." Confirming requires the key holder to pull and decrypt on a laptop after the next scheduled upload, which can be up to 2 hours away. A heartbeat doesn't upload. The box has no lights. In practice the volunteer stands at a winter fridge for up to 2 hours or leaves without confirmation, and H1 and H2 then go unseen until the weekly pull.
- *Cheapest fix:* Add an install mode. For the first 24 hours after provisioning, upload at every heartbeat and after any burst of 10 door cycles. The key holder runs `latchwing pull` while on the phone with the volunteer. For a quicker signal, add one LED that is active only in install mode, inside the opaque box, so the appearance after install doesn't change.

**H4. Nobody checks the Wi-Fi at install, and the fallback is itself a site visit.**
- *Claim:* Guest Wi-Fi reaches at -75 dBm or better. After 24 hours without an ack, the device also scans for a `latchwing-pull` phone hotspot.
- *Failure:* The install sheet has no Wi-Fi step at all. The box is screwed to a post wherever the shelter allows, possibly farther from the building than the fridge. A captive portal or a weak signal shows up only as the 6-hour silence alarm, after the volunteer has left. The hotspot fallback is a good idea, but it needs someone standing at the fridge with a phone, which is a site visit. The proposal concedes the alarm "becomes useless" in this case.
- *Cheapest fix:* Take Scrappy's step 1, and strengthen it: "Stand where the box will go, join the guest Wi-Fi, and open a web page. If a sign-in screen appears or the page won't load, stop." Combined with install mode (H3), the first upload proves the link before the volunteer leaves.

**H5. The report doesn't name the pantry decision it serves, and the key holder carries all the work.**
- *Claim:* Every week a key holder runs a CLI with a passphrase-protected key, builds hourly, daily, and weekly counts, and publishes the buckets. If the key holders can't use a command line, "v1 slips a week."
- *Failure:* Each month, two or three people each do a passphrase-gated CLI run and then decide on their own what to publish and where. The proposal never says who on the pantry side looks at the output, or what they change because of it. A month of correct buckets that nobody acts on fails principle 1 and the "answers a real pantry question" criterion (weight 2).
- *Cheapest fix:* Name the decision now, for example "which day and time the restock run arrives." Have `latchwing report` produce one fixed page for it: busiest 3-hour windows by weekday, hours when the fridge door goes quiet, and gaps shaded. Name who receives it, such as the restock coordinator, and how (shared folder or email).

### Worth stealing
**The hourly heartbeat with a raw door level and a sequence number, plus a gap rule defined entirely by the data: no heartbeat in the hour, or a break in the sequence, means a gap.** This turns "who notices when it stops working" from a guess into a property of the data, at 1-hour resolution instead of Scrappy's 6. It costs about 0.03 mAh a day. `door_open_long`, `low_batt_skip`, and `overwrote` give a key holder plain-language reasons for each gap, which they can pass on to volunteers. Keep it, and add the stuck-closed rule from H1.

### Unchecked claims
- **PCF8523 as a "low-drift clock chip":** checked. Adafruit's listing calls it "not a high-precision device" that "may lose or gain up to 2 seconds a day." That is still much better than the C3's RC, and plenty for 2-hour ack syncs, so the part is fine. But the proposal doesn't say which timer drives the "top of every hour" heartbeat. If it's the C3's own RC timer, the RTC chip doesn't help. Wire the PCF8523 INT pin to a wake-capable GPIO. The listing also says no CR1220 coin cell is included. That's fine if the chip runs on the 3.3 V rail, but say so.
- **Power path:** not specified. Three L91s (up to 5.4 V fresh) can't go to the BAT pads or straight onto 3V3. The Seeed wiki requires a diode for 5V-pin input, and there is none in the BOM. Add the Schottky and state the path.
- **Buffer "covers 78 days at 250 per day":** the power budget itself assumes 400 events a day, which gives about 47 days. The claim that at 5x traffic "the buffer still holds more than 15 days offline" is wrong: 2,000 events a day against about 19k records is about 9.5 days. That's still enough if uploads work, but it matters in exactly the no-Wi-Fi case in section 9.
- **Total ~$54.55:** arithmetic checks. The overrun is $18.23 for the Hammond 1554C2GY, priced from a "regional page; unverified for US." The brief treats $50 as a default and cost carries weight 1, so this isn't disqualifying. But confirm the US price and check that the 3xAA holder, XIAO, and RTC fit before committing.
- **Hamlin 59140 sensor with 57140 actuator through a steel-door spacer at a gap under 10 mm:** unverified. Check the operate distance on a steel offcut at freezer temperature during the key holder's pre-test, which Hardened should borrow from Scrappy.
- **Power arithmetic:** checks. The sum is 6.58 mAh/day, which gives about 175 days at 0.5 derating. Power isn't the risk, as the proposal says.
- **Cloudflare Worker and D1 free tier:** unverified. At this volume, quota limits probably don't matter. The silence-alarm email is the part to verify (see the same item under Scrappy).
