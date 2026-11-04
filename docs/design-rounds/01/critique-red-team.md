# Critique: Team Red, round 01

Reviewer for Team Red. Charter: `docs/teams/red-team.md`. Proposals reviewed: Scrappy, Hardened, Pantry.

All three already get the basics of 0006 right. Batches are sealed to holder public keys on the device, the server stores only ciphertext, acks are signed, debug is locked, and none of them runs BLE or an access point. A stolen board takes lab-grade fault injection to attack (Espressif AR2023-007 covers voltage glitching plus side-channel on both the C3 and the C6). Even then it yields the device identity key and the guest Wi-Fi login, not sealed batches. The data key lives only in RAM, and RAM is lost when the batteries come out. That is a lot of effort for little harm, so I spend no findings on it. The findings below are where a real person with little effort can hurt the data or the trial.

Three gaps are shared by all three proposals. Each is worked out in full once and referenced elsewhere:
- **A. Stuck-closed reads as a real zero** (full write-up under Hardened, finding 2).
- **B. A forged ack deletes data, and the ack's clock is never sanity-checked** (Hardened, finding 1).
- **C. Removing a key holder through their "server token" is server trust, not cryptography** (Pantry, finding 3).

---

## Scrappy

### Biggest risk
**Every timestamp depends on an unbounded clock value from the Worker and on an uncompensated RC oscillator.** The XIAO C3 has no 32 kHz crystal, so between 6-hour syncs the clock runs on the internal RC, and at each sync it jumps to whatever time the ack carries. Two things go wrong:
1. **Cold drift.** Unmeasured cold drift moves events across hour boundaries. The 5 percent check is two 2-hour tallies, so a few minutes of error at each window edge shifts enough events to fail it.
2. **Forged time.** Anyone who holds the ack key (a breached Cloudflare account, or a bad Worker deploy) can move the device clock by hours. Nothing on the device rejects the jump.

The proposal's only guard is to flag batches more than 5 minutes off. That flag cannot tell drift from tampering, and it turns flagged batches into gaps. Accuracy and gaps are the two things the trial is scored on.

### Findings
1. **The identity key is made from pseudo-random output, possibly before storage is encrypted.**
   - **Claim:** "On first boot the device generates its Ed25519 identity key with the RNG … Then burn eFuses: … flash encryption."
   - **Failure:** ESP-IDF's own docs say the C3 RNG is only truly random while the RF is on or `bootloader_random_enable()` is active. The bootloader turns the entropy source off before the app starts, so a first-boot app with Wi-Fi off gets pseudo-random output. Separately, flash encryption does not cover NVS; NVS needs its own NVS-encryption key partition. If the identity key sits in plain NVS, a flash dump of a stolen box yields it. The holder public keys are public, so anyone with the identity key can upload valid sealed batches of invented events and heartbeats. The server accepts them as real, and the liveness email stays quiet even after the box is gone. Scrappy noticed the entropy problem for the batch key but not for the identity key.
   - **Fix:** Turn on flash encryption and secure boot first. Generate the identity key with `bootloader_random_enable()` on, or with Wi-Fi up. Store it in NVS with NVS encryption on. Add the `access-control` check that a flash dump shows no usable key.
2. **A forged ack destroys data (gap B).**
   - **Claim:** the device "advances its read pointer only on a valid ack."
   - **Failure:** The ack signing key is a Worker secret in the same free Cloudflare account that holds R2. One account takeover (a reused password, no 2FA) gives the attacker three things at once: they can delete every stored blob, ack and discard every new upload, and set the clock (see the biggest risk). The device then holds nothing a USB pull could recover.
   - **Fix:** An ack only moves the upload pointer, and only ring overwrite deletes records. The ring is 1.5 MB against 1.1 MB needed, which is about 80 days at 100 opens per day. `latchwing pull --usb` should be able to re-read acked records. Require 2FA and two owners on the Cloudflare account. Have the device reject an ack whose time moves the clock by more than a set bound (say 10 minutes per 6 hours), and log that as a health row.
3. **Stuck-closed reads as a real zero (gap A).**
   - **Failure:** A community fridge door collects fridge magnets. One stuck beside the reed holds it closed. Health rows keep arriving, so every bucket is "data" with zero opens. The 1-hour long-open flag never fires, because the door reads closed.
   - **Fix:** See Hardened, finding 2.
4. **Taking the box costs one second of effort.**
   - **Claim:** "the box is held on by magnets, so a yank pops it off."
   - **Failure:** The attack exposes nothing, and Scrappy's answer there is correct. But the harm is a lost door for the rest of the trial, and the effort is zero. A curious passer-by can lift it off, look, and drop it, and the torn sensor wire then reads open forever. This is the cheapest attack on any of the three designs, and it targets the trial rather than the data.
   - **Fix:** Use VHB plus the magnets, or a short steel tether to the fridge hinge bracket. Then removal takes a tool and intent, not curiosity.
5. **The serial console is not closed.**
   - **Claim:** the device "prints the public key once." The proposal lists USB-JTAG off and secure download mode, but says nothing about the console, BLE, or an access point.
   - **Failure:** A release build that leaves app logging on UART or USB-CDC lets anyone with a cable in the open box read boot logs. Any later debug print of Wi-Fi state or a batch ID leaks into those logs.
   - **Fix:** Print the public key only in a provisioning build. The trial build sets the console to none and states "no BLE, no AP, no console," as Hardened does.

### Worth stealing
**Draw the batch key right after an upload, while Wi-Fi is still on.** This fixes the C3 entropy pitfall at no cost and needs no `bootloader_random_enable()`, which ESP-IDF says conflicts with ADC and Wi-Fi use. It also makes the upload the batch boundary. That caps the open-batch window (the only exposed data under 0006) at 6 hours. Spelling out that "RAM only" means RTC slow memory, which a cold boot wipes, is also the clearest answer any team gave to "what can a thief read."

### Unchecked claims
- **Deep sleep 44 µA:** checked. Seeed-derived datasheets list 44 µA for the XIAO C3. Board-level draw with the diode and regulator is still unmeasured, as the proposal says.
- **"The ESP32-C3 has all four" security features:** true. Note that Espressif advisory AR2023-007 shows secure boot and flash encryption bypass on the C3 via voltage glitch plus side-channel. That is lab-grade effort and exposes only the identity key and Wi-Fi login, which is acceptable if finding 1 is fixed.
- **RNG behavior:** checked, and correct for the batch key. The same rule is what breaks the identity key (finding 1).
- **Free-tier Worker and R2 limits:** unverified. They don't matter at this volume.
- **RC clock drift:** not stated at all, and it matters most. Measure it in the freezer test before shipping.

---

## Hardened

### Biggest risk
**The install check cannot work as written, and the obvious workaround breaks sealing.**
- **Step 4 as written:** "The key holder confirms by phone that the server shows 20 events after the next upload."
- **Why it can't work:** The server cannot see events. That is the whole design. The box has no lights, and uploads run every 2 hours. So the volunteer either stands at a January fridge for up to 2 hours or leaves unverified.
- **What happens next:** A misaligned magnet, which is the proposal's own predicted first failure, is first seen at the weekly pull. That is up to 7 of 30 days lost, or an early site visit.
- **The tempting fix breaks sealing:** have the Worker report a plaintext record count per batch. That puts per-2-hour door counts on the server and undoes the padding.
- **Fix:** For 10 minutes after power-on, upload each sealed batch at once. The key holder runs `latchwing pull` on their phone or laptop and decrypts locally to confirm the 20 events. No plaintext count ever reaches the server.

### Findings
1. **A forged ack deletes data and sets the clock (gap B).**
   - **Claim:** "A breached server can forge acks, which would clear data before anyone pulls it." The device also "sets its RTC from the time in that ack."
   - **Failure:** Hardened names the risk and stops there. A breached Worker can do two things. It can ack every batch, so the 78-day buffer empties and R2 holds nothing. It can also stamp an ack with a time 6 hours off, so a whole day lands in the wrong hour buckets. Sequence numbers would show the deletion, but only after the data is gone. This is exactly my charter's challenge to 0003: right now a forged message can clear data.
   - **Fix:** This costs nothing in hardware, because the buffer already holds 78 days. An ack advances an upload pointer, and only ring overwrite deletes records. A hotspot or USB pull can re-read acked batches. Ack time is accepted only within a bound of the PCF8523's own time (it has no reason to jump more than seconds in 2 hours). A larger jump is logged as `clock_reject` and not applied.
2. **Stuck-closed reads as a real zero (gap A, shared by all three).**
   - **Claim:** "An hour with a heartbeat and no events is a real zero." The hourly raw-level read catches `missed_edge`, and `door_open_long` catches a cut cable.
   - **Failure:** Every tamper signal looks for "open." Nothing looks for "closed too long." Two scenarios:
     - A prankster, or the fridge's usual collection of stickers and magnets, puts a magnet on the frame next to the potted reed.
     - The VHB'd actuator magnet shifts on the steel door and settles near the sensor. That is Hardened's own first-failure scenario.

     Either way the reed reads closed for days. Heartbeats arrive and the raw level agrees with the last state, so there is no `missed_edge`, and every hour is reported as a true zero. A door that averages about 17 opens an hour by day becomes a quiet door in the published chart, and nobody notices. This is also the cheapest way to feed fake data in: here, a fake *absence* of events.
   - **Fix:** The report tool flags "closed with zero opens for more than N daytime hours on a door whose median is above M" as *suspect*. Suspect buckets are shown as gaps, not zeros. The heartbeat row gains `hours_since_last_open`. No new hardware is needed.
3. **The identity key's entropy and storage are not specified.**
   - **Claim:** "The device generates its Ed25519 identity key itself … Then the key holder enables flash encryption."
   - **Failure:** This is the same sequence as Scrappy's finding 1. `bootloader_random_enable()` is specified only for batch keys. If the identity key is drawn without it, or kept in NVS without NVS encryption, a stolen box yields a key that can upload forged sealed batches. Hardened's own "test flash dump must show no plaintext" would catch the storage problem, but only if the dump is taken after the key is written and the NVS partition is inspected.
   - **Fix:** Enable entropy explicitly for identity-key generation. Use NVS encryption. Run the flash-dump test on a fully provisioned unit.
4. **The entropy source fights the battery ADC.**
   - **Claim:** "At each batch start, `bootloader_random_enable()` draws a 256-bit key." The same firmware also reads the battery under load on the ADC before uploading.
   - **Failure:** ESP-IDF says that call is unsafe while the ADC or RF is in use, and that it must be disabled before either is used. If a batch boundary falls next to the pre-upload battery read, the result is either a bad `battery_mv` (which feeds `low_batt_skip`, so uploads get skipped wrongly) or a key drawn while the entropy state is wrong.
   - **Fix:** Use Scrappy's approach: start each new batch right after an ack, while Wi-Fi is up, and drop `bootloader_random_enable()` from the runtime path.

### Worth stealing
**Sequence numbers in the sealed envelope, with any break shown as a gap.** This is the only idea in the round that makes server-side tampering visible from the data itself. A breached server that drops, replays, or reorders batches leaves a hole the key holder sees, without trusting the server. Logging `overwrote` with the lost range does the same for buffer wrap. The offline hotspot scan runs on the hourly heartbeat, not on door activity, which is the right call (compare Pantry's biggest risk). Every team should adopt both.

### Unchecked claims
- **"Low-drift" PCF8523:** not temperature-compensated, and accuracy depends on the crystal. I found no drift figure. With 2-hour syncs it is fine. With the ack time bound in finding 1, its drift sets how tight the bound can be, so measure it in the freezer.
- **C3 "RC clock drifts badly in the cold":** plausible but unsourced. It is the reason the RTC is there, and it matters for Scrappy too.
- **Hammond 1554C2GY at $18.23:** the proposal marks this "regional page; unverified for US." It is the one line pushing the build $5 over budget, so check the US price before treating the overrun as fixed.
- **3000 mAh L91 from "retailer listings":** this is conservative against Energizer's 3500 mAh low-drain rating, so it is safe.
- **Breached server "cannot read data":** true as designed. But "rejects unknown devices" protects nothing if an identity key leaks (finding 3).

---

## Pantry

### Biggest risk
**The rescue-hotspot behavior turns the device into a live door-closed signal, which is a hard-constraint failure.**
- **What the design says:** after 24 hours without guest Wi-Fi, the device tries the rescue network "once per door close and at most every 10 minutes."
- **Over the air:** Each attempt is a Wi-Fi scan and probe from a fixed MAC, a few seconds after the door shuts. Anyone with a phone running a Wi-Fi scanner app sees each visit in real time.
- **On the status page:** When the rescue hotspot is up, each attempt becomes a check-in, and the status page shows "last check-in" for each device. The page then reads, to the minute, when someone last closed the fridge. The install step already relies on this ("checked in just now").
- **Who can trigger it:** An attacker can force rescue mode by jamming or deauthenticating the host's guest Wi-Fi for a day. Most guest networks do not protect management frames. The brief bans live "door is open" feeds outright, and privacy is pass or fail.
- **Fix:** Run rescue attempts on the 6-hour heartbeat timer, or hourly after 24 hours offline as Hardened does, never on a door event. The status page shows check-ins only to the 6-hour window, never as a timestamp. Confirm the install with a key holder's local decrypt, not the status page. Install mode then only needs its LED.

### Findings
1. **The re-provisioning path either doesn't exist or is an open door.**
   - **Claim:** removing a holder means "re-provision with the new key list … a 15-minute visit with a laptop and USB-C." The same board also has "Secure Boot v2, flash encryption, and JTAG and USB-JTAG disable" burned in.
   - **Failure:** In release-mode flash encryption, ESP-IDF blocks plaintext serial reflashing, and OTA is the supported update path. Pantry specifies no OTA. That leaves two cases:
     - (a) The site visit can't change the key list. Removal never happens on this board, and on a one-year battery cycle a removed holder stays a recipient for up to a year.
     - (b) The firmware accepts a new key list over USB. Then anyone who opens the 4-screw lid (the design invites volunteers to open it) can plug in a laptop, add their own public key, and read every future batch.
   - **Fix:** State which case it is. If (b), the provisioning command must be signed by an offline admin key held by two holders, and the device refuses unsigned key lists. Otherwise, re-key by swapping in a freshly provisioned board, as Scrappy does.
2. **Every holder's backup sits in one envelope.**
   - **Claim:** each holder makes "a printed paper backup, which goes into a sealed envelope held by the coordinator."
   - **Failure:** The coordinator is also a key holder, so one person now holds every key: their own, plus a backup of everyone else's. Two scenarios:
     - A burglary at the coordinator's home, or a coordinator who leaves on bad terms, takes all of them. The two-to-three-holder spread in 0006 collapses to one person.
     - Removing the coordinator now forces every holder to rotate keys, not just the coordinator.

     The proposal also doesn't say whether the paper holds the raw key or the passphrase-protected file. If it holds the raw key, the passphrase protects nothing.
   - **Fix:** Each holder keeps their own backup in a different place. Print only the passphrase-protected file.
3. **Same-day removal through the server token is server trust (gap C, all three).**
   - **Claim:** "Revoke the person's server token today, so they can no longer download." Scrappy and Hardened make the same claim.
   - **Failure:** The brief requires that a breached server not expose sealed data. A token only stops a removed holder while the server is honest and they don't control it. The person most likely to be removed, or to leave, is the "tech volunteer," and that is likely the person who opened the Cloudflare account. Their private key still opens every new batch until the key list changes, so with account access they keep reading. Pantry's year-long cycle stretches that window from 30 days to 12 months. For Scrappy and Hardened the window is the trial length.
   - **Fix:** Removal means a re-key within days: a board swap, or a signed key-list update (finding 1). The Cloudflare account gets two owners, neither of them the only tech person, and its access is reviewed when a holder is removed.
4. **Stuck-closed reads as a real zero (gap A).**
   - **Claim:** "An hour between two heartbeats with no door events is a true zero."
   - **Failure:** Pantry's 6-hour heartbeat makes this worse than Hardened's. A magnet left on the frame makes days of real traffic look like true zeros. Those zeros feed directly into the restock-time decision this proposal says it exists to change ("Saturdays are emptied by 2 p.m.").
   - **Fix:** See Hardened, finding 2.
5. **Identity-key entropy is not specified.**
   - **Claim:** `latchwing provision` "generates the Ed25519 identity key on the chip."
   - **Failure:** The C6 has the same RF-or-bootloader entropy rule as the C3. Provisioning runs over USB with no Wi-Fi up, so this is Scrappy's finding 1 again. Pantry does get the batch key right.
   - **Fix:** Enable entropy explicitly, use NVS encryption, and run a flash-dump test.

### Worth stealing
**The report tool decrypts in memory and never writes raw rows unless asked with `--raw`.** "A key holder loses a laptop" is the most likely real breach in this whole project: far more likely than a fault-injected chip. This cuts its harm from up to 90 days of raw events to nothing beyond the passphrase-protected key file. It costs a few lines of code, and Scrappy's and Hardened's tools should adopt it. The outside label with a QR code to the published totals comes second: showing people the output is the strongest answer to "this looks like surveillance."

### Unchecked claims
- **Deep sleep 15 µA (vendor forum figure):** partly checked. Seeed-derived listings and one forum measurement give 14–16 µA. Another C6 board measured about 667 µA because of board-level leakage. The 728-day life and the year-long service plan depend on this number, and the trial does not. Measure it, and drop the yearly cycle from the trial claims until it is measured.
- **"The ESP32-C6 has … everything 0006 requires":** true. AR2023-007 (glitch plus side-channel bypass of secure boot and flash encryption) also covers the C6. That is lab-grade effort and acceptable.
- **The rescue retry budget (one 15 s attempt a day):** this does not match the per-door-close rescue behavior. At 50 visits a day, capped at one attempt every 10 minutes, it is roughly 50 × 15 s × 100 mA ≈ 21 mAh a day, about 11 times the whole budget. That is fine for 30 days but cancels the year-long claim. The fix in the biggest risk removes this too.
- **SM-200Q/W gap on a steel door, and the $18 SM-226L-3Q fallback:** unverified. This matters for accuracy, not security.

---

Sources checked: [Espressif ESP32-C3 RNG docs](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/api-reference/system/random.html), [ESP32-C3 flash encryption docs](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/security/flash-encryption.html), [ESP-IoT-Solution flash encryption note](https://docs.espressif.com/projects/esp-iot-solution/en/latest/security_solution/flash_encryption.html), [Espressif AR2023-007 (C3/C6)](https://documentation.espressif.com/AR2023-007%20Security%20Advisory%20Concerning%20Bypassing%20Secure%20Boot%20and%20Flash%20Encryption%20using%20CPA%20and%20FI%20attack%20on%20ESP32-C3%20and%20ESP32-C6%20EN.html), [ESP-IDF v3.2 flash encryption on NVS](https://docs.espressif.com/projects/esp-idf/zh_CN/release-v3.2/security/flash-encryption.html), [esp32.com NVS encryption thread](https://esp32.com/viewtopic.php?p=110997), [XIAO ESP32C6 sleep figures](https://wiki.openelab.io/seeed-studio/xiao-esp32-c6-dev-board), [C6 board-level sleep leakage](https://esp32.com/viewtopic.php?p=152254).
