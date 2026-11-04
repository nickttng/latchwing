# Revision: Team Scrappy, round 01

## 1. Responses

**Hardened**
- **H-risk / H1, cold VHB on steel: accept.** 3M's 50 °F floor applies to the steel, and a hand warmer can't hold it there. Fix: substrate at 50 °F or more, measured with an IR thermometer, is a pass/fail install step. We install on a mild November afternoon instead of bonding cold. The cabinet is screwed. We also switch to a branded contact with a published gap (Seco-Larm SM-200Q) and bench-test it on a steel offcut at freezer temperature.
- **H2, no chatter limit, silent overwrite: accept.** The arithmetic holds: a magnet at the edge can drain the cells in about 4 days. Fix: more than 20 wakes in 1 minute writes a `chatter` row and turns pin wake off for 10 minutes, and the report shows that window as a gap. The health row gets an `overwritten` count.
- **H3, stuck closed reads as quiet: accept.** Health rows now carry the raw pin level, wakes since the last row, and `hours_since_open`. The report marks long daytime silence as "suspect" and shows it as a gap (section 4).
- **H4, power cut mid-write: accept.** Records are framed as length + sequence + AEAD record, and the tag is the commit check. Boot scans to the last valid frame and logs `torn`. Pointers live in NVS. We add a 470 µF polymer cap and `low_batt_skip`, and the bench test cuts power mid-upload in the freezer.
- **H5, a yank loads the sensor: accept.** A strain-relief clip goes within 5 cm of the sensor, with a short drip loop. The box now uses VHB as well as magnets, which the warm-install rule makes possible.
- **Unchecked: the RC drift makes the 5-minute flag meaningless: accept.** We drop the flag and correct drift downstream instead (Red-risk below).

**Pantry**
- **S1, no user LED: accept.** This was a real error. Fix: a 3 mm LED and 330 Ω resistor on D10, lit only in install mode. Install mode also uploads right away, so the first ack and the key holder's live decrypt both happen while the volunteer is still at the fridge.
- **S2, cold-boot key entropy: accept the finding, reject the fix.** ESP-IDF warns that `bootloader_random_enable()` conflicts with the ADC and RF, which is Red's point against Hardened. Fix: every key, cold boot included, is drawn after `esp_wifi_start()`, with the RF on whether or not an AP joins, and before any ADC read. Test: two cold boots in a row give different wrapped keys.
- **S3, no Wi-Fi fallback: accept.** We add a second SSID, a key holder's phone hotspot. After a missed window it is tried on the hourly timer only, never on a door event. Install step 1 now checks for a captive portal. We also ask the host to warn the pantry before changing the password. A hotspot rescue is still a visit, but it never opens the box and loses no data.
- **S4, the box gets pulled off: accept.** Same fix as H5.
- **S5, a fallen magnet sticks to the frame: accept.** Fridge door: 4 or more hours between 06:00 and 22:00 with no opens is suspect. Cabinet: 24 hours. Both are tuned after week 1.
- **Unchecked notes: all accepted.** The memory is RTC fast memory (`RTC_DATA_ATTR`), not slow. Each holder confirms their Worker email address before the trial. The firmware learns the closed-door level at install instead of trusting "NO/NC" on the label.

**Red team**
- **Red-risk, RC drift and an unbounded ack clock: accept, but no RTC chip.** Uploads move from every 6 to every 3 hours. Each ack records device time against server time. The report spreads drift linearly between syncs, so only drift that changes speed within 3 hours is left over. An ack that moves the clock by more than 5 minutes is refused and logged as `clock_reject`, and that window shows as suspect. Why no RTC chip: with 1 minute of leftover error at each edge of a 2-hour tally window, at about 10 opens an hour, about 0.3 of 20 events move, roughly 1.5 percent. The freezer test measures the real number; the bench-test gate is 60 s of leftover error per 3 hours.
- **R1, the identity key's entropy and storage: accept.** On first boot, flash encryption and NVS encryption (HMAC eFuse scheme) are already on. The key is generated with the RF on and stored in encrypted NVS. The flash-dump check runs on a fully provisioned unit.
- **R2 / gap B, a forged ack deletes data: accept.** An ack only moves the upload pointer. Records go only when the ring overwrites them, and each overwrite is logged. At 100 opens/day the ring holds about 80 days, so a USB or hotspot pull can still re-read records that were acked. The Cloudflare account gets two owners with 2FA, and neither is the only tech volunteer.
- **R3 / gap A: accept** (see H3 and S5).
- **R4, taking the box costs one second: accept.** VHB plus magnets means removal takes a tool and intent.
- **R5, open serial console: accept.** The trial build sets the console to none. The public key is written once through the USB-serial driver during enrollment, and never again once the first signed ack arrives. The driver accepts one input command, `dump-sealed`, which streams sealed frames and nothing else. No BLE, no AP.
- **Gap C, removal by token is server trust: accept.** Removal is now a re-key within 7 days, by swapping in a freshly provisioned board. Deleting the token is a courtesy, not the control. Account access is reviewed at every removal.

## 2. Stolen ideas
- **Hardened:** the hourly heartbeat, sequence numbers in the envelope (a break shows as a gap), `low_batt_skip`, the 470 µF cap, padded uploads, and the two-SSID fallback.
- **Pantry:** the Seco-Larm SM-200Q contact; the report decrypts in memory and writes no raw rows unless run with `--raw`; a paper backup of each holder's passphrase-protected key file, kept by that holder in a separate place (with Red's fix); and naming the pantry decision.

## 3. The design now

### 3.1 The design in three sentences
Each door gets its own copy of the same build: a XIAO ESP32-C3, a Seco-Larm SM-200Q reed contact, an install-only LED, and three L91 lithium AAs, in an IP65 box held on with VHB and magnets, installed on a day when the steel is at 50 °F or warmer. The chip sleeps until the door changes state, limits chatter, writes a sealed health row every hour, and every 3 hours uploads padded, sealed batches over guest Wi-Fi (with a phone-hotspot fallback) to a free Cloudflare Worker that can store them but not read them. A key holder's laptop pulls and decrypts in memory every morning, flags suspect windows, and publishes hourly counts with gaps shown, to help decide when restock drops happen.

### 3.2 Parts (per device)
| Part | Price | Source |
|---|---|---|
| Seeed XIAO ESP32-C3 | $4.99 | seeedstudio.com, checked 2026-11-03 |
| Seco-Larm SM-200Q/W surface contact, 3/4" gap | $2.71–5.95 | JMAC / FPC, price checked by Pantry; we did not re-check |
| 3x Energizer L91 AA | ~$9 | Walmart, **unverified today** |
| IP65 ABS box, 100x68x50 mm | $10.99 | Newegg Business |
| 3xAA holder, PG7 gland, 1N5817, 3x 1 MΩ | ~$3.20 | **unverified** |
| 470 µF solid-polymer cap, -55 °C rated | ~$1 | DigiKey, **unverified** |
| 3 mm LED + 330 Ω resistor | ~$0.10 | parts bin, **unverified** |
| Silica gel 5 g, label | ~$0.80 | **unverified** |
| 2 rubber-coated cup magnets, VHB 5952, adhesive cable clip | ~$4.30 | **unverified** |
| IR thermometer, one per two-device kit | ~$5 share of $10 | **unverified** |
| **Total** | **~$42–45** | Under $50. Server: $0, free tier, **unverified** |

No custom board, and every part is in stock in the US.

### 3.3 Power (`skills/enclosure-and-power`, 100 opens/day = 200 events)
| State | mA | h/day | mAh/day |
|---|---|---|---|
| Deep sleep (Seeed 44 µA) | 0.044 | 24 | 1.06 |
| Regulator + diode leakage (**unverified**, to be measured) | 0.020 | 24 | 0.48 |
| Reed pull-up + battery divider | 0.0058 | 24 | 0.14 |
| Door wake, debounce, seal, log: 200 x 0.25 s | 25 | 0.0139 | 0.35 |
| Hourly health row: 24 x 0.2 s | 25 | 0.0013 | 0.03 |
| Upload, sign, ack verify: 8 x 10 s | 110 | 0.0222 | 2.44 |
| **Total, normal** | | | **4.50** |

Life = 3000 mAh x 0.5 / 4.50 / 1.3 = **~256 days at freezing.** Bad cases:
- **Wi-Fi down:** 8 failed 15 s tries plus 24 hourly 2 s hotspot scans makes 6.93 mAh/day, about **166 days**.
- **Chatter at the limiter's cap:** about 2,600 wakes a day makes 8.70 mAh/day, about **133 days**.
- **Sleep current 10x the spec:** about **82 days**.

None of these threaten 30 days. The risk is a voltage sag under TX peaks in the cold, not capacity. The cap carries the peaks. Below 3.6 V on the cells under load, the device skips the upload and logs `low_batt_skip`.

### 3.4 Behavior
- **Wake:** the reed on D1 wakes the chip on the level opposite its current state. Then a 50 ms settle, a re-read, and an event only if the state changed, with a 500 ms lockout. The closed-door level is learned during install mode.
- **Chatter limit:** more than 20 wakes in 1 minute writes a `chatter` row and turns pin wake off for 10 minutes. The report shows that window as a gap.
- **Seal:** each batch key comes from the RNG after `esp_wifi_start()`. It lives in RTC fast memory, survives deep sleep, and is lost on power-off. A new batch starts at every ack and at every cold boot. Holder wrapping uses libsodium `crypto_box_seal`, and records use ChaCha20-Poly1305 with a counter nonce.
- **Store:** a 1.5 MB flash ring of `[len][seq][AEAD]` frames. The upload and write pointers are in NVS. Boot scans to the last valid frame and logs `torn`. The ring only overwrites when full, and logs `overwritten` with the range lost.
- **Health row, hourly:** `battery_mv` under load, `boot` and reset reason, `pin_level`, `wakes`, `hours_since_open`, `rssi`, `bounce_rejects`, `chatter`, `overwritten`, `torn`, `clock_reject`, and `low_batt_skip`.
- **Upload:** every 3 hours on a fixed timer, padded to size steps, plus an overflow guard at 50 percent full. The upload is signed with the identity key, and the Worker returns a signed ack with device, batch, and time. The ack moves only the upload pointer. Clock offsets are logged for correction downstream. A jump of more than 5 minutes is refused. After a missed window, the device scans for the hotspot SSID on the hourly timer.
- **Install mode, the first 10 minutes after power-on:** the LED blinks on each change, and each batch uploads right away. Three long blinks mean an ack came back. After that the LED is off for good.
- **To a person:** the Worker emails the holders after 12 hours of silence, from metadata only. Each morning a scheduled `latchwing pull && latchwing report` runs on one holder's laptop. It decrypts in memory and checks sequence numbers. It applies the gap rules and the suspect rules: fridge 4 or more daytime hours with no opens, cabinet 24 hours, any `chatter`, any `clock_reject`. It then emails that holder one line: OK, or which rule fired. Every week, a holder posts a fixed page to the shared folder: busiest 3-hour windows by weekday, the hours when the fridge goes quiet, and gaps shaded. The restock coordinator uses it to set drop times.

### 3.5 Install and upkeep
**Bench, done by a key holder:** provision, run 24 hours in a freezer, cut power during an upload, and run 20 tallied opens. Measure drift and sleep current, and check the contact's gap on a steel offcut at freezer temperature.

**Site, done by a volunteer from a one-page card, on an afternoon forecast at 50 °F or warmer:**
1. Stand at the fridge, join the guest Wi-Fi on your phone, and open a web page. If a sign-in screen appears or the page won't load, stop.
2. Clean with the alcohol wipe. Check the steel with the IR thermometer. Below 50 °F, stop and pick another day.
3. Stick the contact on the frame and the magnet on the door, at the template marks with the gap under 10 mm. Tug the magnet. If it moves, stop. Clip the lead 5 cm from the contact, with a drip loop.
4. Put the box on the side wall with VHB and magnets, gland down, low on the wall. Cabinet: screw both the parts and the box.
5. Put the batteries in. Open and close the door 10 times and count 20 blinks, then wait for three long blinks. Phone the key holder, who decrypts and confirms 20 events.
6. Close the lid, stick on the notice, and send a photo.

**Upkeep:** none planned. If a morning email names a suspect window, a volunteer re-checks the magnet; that is a visit, and it gets counted. Two 2-hour manual tallies happen during the trial.

### 3.6 Data access (`skills/access-control`)
- **Who:** 2 or 3 named holders, names kept outside the repo. Each has an X25519 key file protected by a passphrase. Each prints a paper copy of their own protected file and keeps it in a different place from everyone else's.
- **How:** the server, the network, and a USB `dump-sealed` see only sealed frames. The report never writes raw rows unless run with `--raw`. R2 deletes blobs after 90 days, and local raw copies follow the same rule.
- **Removal:** a key holder provisions a board with the new list, and it is swapped in within 7 days at two screw terminals. The old board is wiped. The removed person's token is deleted the same day as a courtesy, and the Cloudflare owners and 2FA are reviewed. They keep what they already read.
- **Provisioning:** the firmware is signed (Secure Boot v2), with holder public keys and the ack public key built in. On first boot, flash encryption in release mode and HMAC NVS encryption are enabled first. Then the identity key is generated with the RF on and stored in encrypted NVS. The public key is written once and enrolled on the Worker. eFuses: USB-JTAG and pad JTAG off, secure download mode on. Console none, no OTA, no BLE, no AP. Run the flash-dump check on a fully provisioned unit.
- **Chip:** the C3's RNG with the RF on, Secure Boot, flash and NVS encryption, and the debug lock. No secure element.

### 3.7 Decisions
- **0001 keep, amended.** A timer wake writes health rows only and never creates a door event. The chatter limit can turn pin wake off for 10 minutes, and that window is logged as a gap.
- **0002 keep, amended.** Still four fields. The sequence number sits in the envelope. The clock is the C3's RTC, synced only from signed acks, bounded at 5 minutes per sync, and drift-corrected downstream.
- **0003 change.** An ack advances an upload pointer and deletes nothing. Only ring overwrite deletes, and it is logged. Uploads run on a fixed 3-hour schedule, padded, with the overflow guard kept. Transports: Wi-Fi, a hotspot, and a USB sealed dump.
- **0004 keep, plus suspect rules.** No health row in an hour, a sequence break, chatter, `clock_reject`, or long daytime silence all show as gaps, never zeros.
- **0005 keep.** Mains is still rejected. The install-temperature rule is added.
- **0006 keep, amended.** Removal is a board re-key within 7 days, not at the next provisioning. "RAM only" means RTC fast memory. All keys are drawn with the RF on.

### 3.8 Open questions
1. **Dead vs. quiet:** hourly health rows with sequence numbers. A missing row or a sequence break is a gap. A healthy chip whose sensor has gone silent is suspect, also shown as a gap. The 12-hour silence email and the morning check bring it to a person within a day.
2. **Mains:** still rejected. The owner of the outlet is unknown, a cord is easy to unplug, and the cells give more than 4x the trial even in the worst case above.
3. **One device per door,** for the reasons already given: the distance is unknown, and failures stay independent.
4. **Pulling and looks:** VHB plus magnets needs a tool to remove. The clip keeps a pull off the sensor. A cut wire reads open for a long time and is flagged. A missing box is a gap. The box is plain white, the LED is dark after install, and the notice reads "Door counter. Counts openings only. No camera, no microphone, no tracking."

### 3.9 Assumptions
- **Guest Wi-Fi reaches the fridge with no captive portal.** If wrong, data buffers about 80 days and comes back through the hotspot or USB. We lose liveness, which is the main loss.
- **There is a November afternoon at 50 °F or warmer before the start** (normals not checked). If not, the start slips. We do not bond cold.
- **Steel fridge, wood cabinet.** If the fridge is plastic or aluminum, the box uses VHB only.
- **About 100 opens a day.** At 1,000, life is still about 150 days and the 3-hour uploads keep the ring nearly empty.
- **Two holders, and one laptop that can run a morning task.** If there is only one holder, we fail 0006's minimum and wait.

### 3.10 First failure
A marginal gap on the steel fridge, short of chatter or a stuck reed. Door sag or a knocked magnet leaves the reed at the edge of its switching distance, so it misses some closes. The suspect rules don't fire, unpaired opens get dropped, and the count slips past 5 percent. The morning report shows a rising unpaired rate and `bounce_rejects`, and the first manual tally confirms it. Re-seating the magnet is a site visit.
