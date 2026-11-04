# Revision: Team Hardened, round 01

## 1. Responses
Sources: `critique-scrappy.md` (S), `critique-pantry.md` (P), and `critique-red-team.md` (R), including the shared gaps A, B, and C.

- **Install check cannot work; the server can't see events (S1, P-H3, R biggest risk). Accept.** This was a real error. Replaced with a one-shot install mode: for 15 minutes the device uploads each sealed batch within 30 s of a door change, and a key holder decrypts it locally while on the phone. No count ever reaches the server. **I reject the LED.** An LED proves the sensor but not the Wi-Fi link. A local decrypt proves both, and with an LED the gray box would need a window.
- **The predicted first failure (the VHB magnet) has no fix (S biggest risk, P biggest risk, P-H2). Accept.** The moving magnet is now held mechanically, with VHB only as a backup. The install sheet also gains a tug test. Pulls run daily for the first 3 days, alongside a manual tally.
- **VHB at 40 °F is below 3M's minimum, and a hand warmer can't warm steel (S, P-H2). Accept.** The rule is now a 50 °F *surface*, measured with a thermometer strip. Below that, the clamp carries the load and the tape only holds position.
- **Stuck closed reads as a real zero (R2/gap A, P-H1). Accept.** A peeled neodymium actuator snaps to the steel beside the reed, so this is the most likely way our own first failure would play out. The heartbeat gains `mins_since_change`, and the report marks long daytime zeros as *suspect*, shown as gaps. The rule lives in the report, never on the server.
- **A forged ack deletes data and sets the clock (R1/gap B). Accept.** An ack now moves only the upload pointer, and only ring wrap deletes records. Ack time is bounded against the PCF8523. Recovery is local and does not depend on the server (section 4).
- **Token removal is server trust (gap C). Accept.** The guarantee is now a re-key by swapping in a spare unit within 7 days. Revoking the token is only a same-day courtesy. The Cloudflare account gets two owners with 2FA.
- **Identity-key entropy and NVS storage are unspecified (R3). Accept.** The key is generated with Wi-Fi up during provisioning and stored in encrypted NVS. The flash-dump test runs on a fully provisioned unit.
- **`bootloader_random_enable()` conflicts with the ADC (R4). Accept.** A batch now starts right after an ack, while Wi-Fi is up. The only other entropy call is on cold boot, before the ADC or RF has been touched, and it is disabled before either is used.
- **Release-mode flash encryption with no update path (S2). Reject, with mitigation.** The `access-control` check needs "a flash dump shows no usable keys". Without flash encryption, a stolen box gives up the identity key, and that key can upload forged heartbeats that hide a missing device (R3). That is the silent failure our charter forbids. To manage the risk: a sacrificial unit is locked first and soaked for 7 days (freezer plus outdoors), and the trial carries one fully provisioned spare.
- **The commit byte can't be written in encrypted flash (S3). Partly accept.** Fixed: the ring moves to an unencrypted partition, which is safe because its contents are already AEAD ciphertext. **I keep the commit byte.** After a reboot the data key is gone, so the device cannot check old tags. The commit byte is the only way it can find and log a torn tail by itself, rather than waiting for a pull. The cost: a thief can count records, which is the busyness hint 0006 already accepts.
- **Delete the PCF8523 (S4). Reject.** The clock bound in gap B needs a clock the server cannot set. The PCF8523's quoted 2 s/day (per the Adafruit listing, P) makes a ±60 s bound safe even through a month offline. Accepted from P: its INT pin now drives the hourly wake, and it runs from the 3V3 rail with no coin cell.
- **Upload every 6 h, not 2 h (S5). Accept.** This cuts cold-sag exposure to a third. The alarm now fires after 2 missed windows (12 h). The heartbeat stays hourly.
- **Buffer math is wrong (S, P). Accept.** At 96 B fixed records, the buffer is 19.1k records, which lasts 45 days at 424 records a day and 9.4 days at 5x traffic.
- **No diode in the power path (P). Accept.** Added a 1N5817 into the 5V pin, and the battery is read on the cell side.
- **The report names no decision and no recipient (P-H5). Accept.** It serves restock-drop timing, goes to the restock coordinator, and has a fixed page format.
- **No Wi-Fi check at install (P-H4). Accept.** This is now install step 1.
- **The Hammond price is unverified (S, P, R). Checked.** An aggregator snapshot of DigiKey US dated 2026-06-10 shows $20.21. That is higher, so the overrun grows.
- **"RC drifts badly in the cold" is unsourced (R). Accept.** It is now a freezer measurement in the bench soak, not a claim.

## 2. Stolen ideas
- From Scrappy: the batch key is drawn right after an upload with Wi-Fi on; "RAM only" means RTC slow memory; the phone Wi-Fi check at install; two 2-hour manual tallies.
- From Pantry: the report decrypts in memory and writes raw rows only with `--raw`; the named restock decision; the outside label with a QR code to published totals; each holder keeps their own backup (R's fix to Pantry).
- From Red Team: the ack only moves a pointer, the ack clock is bounded, the suspect-zero rule, removal by re-key, and two Cloudflare owners.

## 3. The design now

### 3.1 The design in three sentences
Each door gets one identical sealed polycarbonate box: an ESP32-C3, three lithium AA cells, a PCF8523 clock that drives an hourly heartbeat, and a potted reed sensor whose door magnet is clamped mechanically rather than only taped. Every event and heartbeat is sealed to the key holders and committed to flash with a sequence number. Uploads run every 6 hours to a server that can neither read the data nor delete it from the device. A report gives the restock coordinator busy and quiet hours per weekday, and it shows dead, missing, and suspect-zero hours as gaps.

### 3.2 Parts (per device; checked 2026-11-03 unless marked)
| Part | Price | Source |
|---|---|---|
| Seeed XIAO ESP32-C3 | $5.39 | RobotShop |
| Adafruit PCF8523 RTC (#5189), on the 3V3 rail, INT to a wake GPIO | $4.95 | adafruit.com |
| Hamlin 59140-1-S-02-A potted reed, -40 °C | $2.48 | Future Electronics |
| Hamlin 57140-000 actuator (screw-mount body) | ~$3 (unverified) | DigiKey |
| Second 59140 reed inside the lid, the one-shot install trigger | $2.48 | Future Electronics |
| Hammond 1554C2GY polycarbonate box, IP68 | $20.21 (aggregator snapshot of DigiKey US, 2026-06-10) | DigiKey |
| 3x Energizer L91 | $9.00 ($23.98 for 8; unverified today) | Walmart |
| 3xAA holder with leads | ~$2 (unverified) | DigiKey/Amazon |
| PG7 gland plus 1 m jacketed 2-core cable | ~$3 (unverified) | hardware store |
| 1N5817 Schottky, 470 µF polymer cap (-55 °C), 1 MΩ resistors, 100 nF caps | ~$2.20 (unverified) | DigiKey |
| Stainless worm-gear clamp, 2 UV-stable cable ties, small aluminum backer for the actuator | ~$2.50 (unverified) | hardware store |
| Silica gel, VHB 5952, 2 tamper screws, label | ~$3 (unverified) | hardware store |
| **Total** | **~$60.21** | |

This is $10 over the default budget. The polycarbonate box is the main cost, and we keep it because ABS turns brittle near -20 °F. Cost carries weight 1, and reliability comes first. The trial needs 2 installed units plus 1 provisioned spare, about $181 before shipping.

### 3.3 Power (`skills/enclosure-and-power`; currents unverified until measured)
| State | mA | h/day | mAh/day |
|---|---|---|---|
| Deep sleep: board ~60 µA, reed pull-ups, divider, RTC | 0.065 | 24 | 1.56 |
| Door wake: 400/day x 150 ms (seal + commit) | 25 | 0.0167 | 0.42 |
| Heartbeat: 24/day x 200 ms | 25 | 0.0013 | 0.03 |
| Upload: 4/day x 10 s (join, TLS, sign, verify ack) | 110 | 0.0111 | 1.22 |
| Failed-upload allowance: one 15 s attempt/day | 100 | 0.0042 | 0.42 |
| Offline hotspot scan (only after 24 h offline): 24 x 1.5 s | 90 | 0.01 | 0.90 |
| **Total (worst case, offline)** | | | **4.55** |

Life = 3000 mAh x 0.5 / 4.55 / 1.3 = **253 days at freezing**, or 152 days at 0.3 derating for -20 °F. The install mode (15 minutes, once) adds well under 1 mAh. Sag remains the real risk. Before each upload the firmware reads the pack under load on the cell side of the diode. Below 3.6 V (unverified, to be set in the freezer test) it skips the upload and logs `low_batt_skip`. Brownout reset is on.

### 3.4 Behavior
- **Door wake (0001):** the reed has an RC filter and wakes the chip on the opposite level. The firmware waits a 50 ms settle, re-reads, and logs only a state change. The 500 ms lockout is spent asleep.
- **Heartbeat:** PCF8523 INT wakes the chip hourly. It logs a health row with `battery_mv`, `temp_c`, the raw level, `mins_since_change`, `buffer_pct`, and the sequence number. It also logs `missed_edge` when the raw level disagrees with the last recorded state, and `door_open_long` after an open lasting more than 30 minutes. It never invents a door event.
- **Write path:** the ring sits in an unencrypted 1.75 MB partition of fixed 96 B slots: `[seq][ciphertext+tag]`, then a commit byte. On boot the firmware logs `torn_write` for an uncommitted tail, plus a `boot` row with the reset reason.
- **Batches:** a new batch key is drawn right after each ack, while Wi-Fi is on. On cold boot the key is drawn with `bootloader_random_enable()` before the ADC or RF is used, then disabled. The key lives in RTC slow memory and is wiped at batch close.
- **Upload:** every 6 hours on a fixed schedule, padded to size steps and signed by the device. A signed ack that names the device and batch moves **only the upload pointer**, and only ring wrap deletes records. A wrap logs `overwrote` with the lost sequence range.
- **Clock:** ack time is applied only if it is within ±60 s of the PCF8523. A larger jump logs `clock_reject` and is not applied. If the RTC lost power (oscillator-stop flag), the first ack sets it, and that is logged as `clock_set`.
- **Server-independent recovery:** after 24 hours without an ack, each heartbeat scans for the `latchwing-pull` hotspot. On it, `latchwing pull --local` runs on a key holder's laptop and presents a challenge signed by a holder key (holder Ed25519 public keys are in the config). The device then sends every record still in the ring, whether acked or not. A breached server can't block or forge this path.
- **Install mode:** a magnet held to the lid mark starts it, once per provisioning and never again. For 15 minutes the device uploads within 30 s of each door change. That leaks door timing over RF only for those 15 minutes, while the installer is the only user.
- **To a person:** the server emails all key holders after 2 missed windows (12 h), using upload metadata only. In week 1 a key holder pulls daily. After that, `latchwing report` runs weekly: it decrypts in memory, writes no raw rows unless given `--raw`, and checks signatures and the sequence. A gap is an hour with no heartbeat or a sequence break. A **suspect** hour is a daytime hour (8 a.m. to 10 p.m.) that falls in a run of 4 or more heartbeat hours with zero events, on a door whose median for that weekday-hour is 3 or more. In week 1, before medians exist, a fixed 4-hour daytime rule applies. A suspect hour is shown as a gap with its reason.
- **The decision:** what time the restock drop happens, and whether the busiest days need a second drop. The report is one page with the busiest 3-hour windows by weekday, the quiet hours, and gaps hatched. It goes to the restock coordinator in the pantry's shared folder, and the published totals are linked by QR from the label.

### 3.5 Install and upkeep
A key holder provisions, freezer-tests, and seals both boxes, and does a 20-open tally. The volunteer:
1. Stands where each box will go, joins the guest Wi-Fi on a phone, and loads a web page. If a sign-in screen appears or the page fails, they stop and call the key holder.
2. Checks a thermometer strip on the door and frame. At 50 °F or above, VHB counts. Below that, the clamp is the hold and the tape is positioning only.
3. Cleans with alcohol. Sticks the sensor on the frame at the latch edge, gap under 10 mm by the paper gauge, with the spacer on steel. Fixes the actuator to the aluminum backer and clamps the backer to the door handle with the worm clamp (fridge), or screws it to the door (wood cabinet). Tugs it firmly, and stops if it moves.
4. Screws the box to the post or wall beside the fridge, gland down, at least 18 in. above the ground and clear of drips. Clips the cable every 6 in. Never drills into the fridge.
5. Calls the key holder, holds the magnet card to the lid mark, and opens and closes the door 10 times. The key holder runs `latchwing pull` and reads back 10 opens and 10 closes. Then repeats for the cabinet.
6. Sticks on the notice: "Door counter. Counts openings only, so volunteers know when to restock. No camera, no microphone, no names. Totals: [QR]. Questions: [pantry contact]."

Upkeep: a volunteer does two 2-hour manual tallies for the 5 percent check, one of them inside the first 3 days. If an alarm fires, or the report shows suspect or `missed_edge` rows, a volunteer checks the box and the magnet. Cells and desiccant are swapped only at re-provisioning.

### 3.6 Data access (`skills/access-control`)
- **Who:** 2 or 3 named key holders, with names kept off the repo. Each has a passphrase-protected X25519 key, plus an Ed25519 key for recovery challenges. Each keeps their own offline backup of the passphrase-protected file, in separate places.
- **How:** per-holder download tokens. Batches stay sealed end to end. The holder decrypts locally and in memory. Local recovery works without the server.
- **Removal:** revoke the token the same day (a courtesy that relies on the server). The guarantee is a swap within 7 days to the spare unit, provisioned without their keys. The old unit is wiped. The Cloudflare account's ownership is reviewed, and it always has 2 owners with 2FA, neither of them the only technical person. Removed holders keep what they already read.
- **Server:** a Cloudflare Worker with D1 (free-tier limits unverified). It verifies device signatures and rejects unknown devices. A breach can drop or stall uploads, but it cannot read data, delete records on the device, or move the clock by more than 60 s.
- **Provisioning and chip:** Secure Boot V2 signed firmware. Config holds the holder public keys, the ack public key, guest and hotspot logins in encrypted NVS, and the door ID. The device brings Wi-Fi up on the bench network, generates its Ed25519 identity key, and stores it in encrypted NVS. Only the public half is printed, and only in the provisioning build. Then: flash encryption in release mode, JTAG and USB-JTAG eFuses burned, secure download mode on, console off, no BLE, no access point. A flash dump of a fully provisioned unit must show no usable keys or logins. A sacrificial unit is locked and soaked for 7 days before the trial units.

### 3.7 Decisions
- **0001: Keep, amended.** Door state is interrupt-only, plus an hourly RTC timer wake for liveness. A heartbeat never creates a door event.
- **0002: Keep, amended.** The 4-field record is unchanged. The envelope adds a sequence number and commit byte. Health rows add `mins_since_change`, `missed_edge`, `door_open_long`, `torn_write`, `overwrote`, `low_batt_skip`, `clock_reject`, and `clock_set`. Time comes from the PCF8523.
- **0003: Changed.** "A batch is cleared only after a signed ack" becomes "an ack advances the upload pointer; only ring wrap deletes; a holder-signed local pull can re-read any record still in the ring". Uploads run on a fixed 6-hour schedule, padded, with no threshold trigger. The buffer covers 45 days at the assumed load.
- **0004: Keep.** A gap is a missing heartbeat or a sequence break, and a suspect zero is also shown as a gap.
- **0005: Hold.** The enclosure stays sealed IP68 on batteries.
- **0006: Hold, amended.** Removal is by re-key (unit swap) within 7 days. A token is only a courtesy.

### 3.8 Open questions
1. **Dead vs. quiet:** an hour with a heartbeat and no events is a zero, unless the suspect rule flags it. An hour with no heartbeat, or a sequence break, is a gap. The alarm reaches people within 12 h.
2. **Mains:** still rejected. The host controls the outlet, and the fridge loses power exactly when data matters most. The cord is the easiest part to pull or freeze, and the battery margin is about 8x.
3. **One device per door,** so failures are independent and there is no cable run of unknown length between doors.
4. **Pulling and appearance:** the box is screwed down and the magnet is clamped. A cut cable gives `door_open_long`, a magnet planted on the frame gives a suspect zero, and a missing box trips the alarm and leaves a gap. A stolen box yields no key. The box is plain gray, with no lights and no lens, and the label says what it is for and links to the totals.

### 3.9 Assumptions
- **Guest Wi-Fi reaches the box at -75 dBm with no captive portal.** Install step 1 checks it. If it is wrong, data arrives only through a local pull. Nothing is lost for 45 days, but the trial fails "no site visit", and v2 needs LoRa.
- **There is a post or wall to screw to.** If not, the box goes on the fridge side with VHB and a strap, and it is easier to pull.
- **The fridge door has a handle that takes a clamp.** If not, the actuator relies on VHB at a surface of 50 °F or more, and the first-3-days pulls become essential.
- **The fridge is steel and the cabinet is wood.** If either is wrong, the spacer and screw choices flip.
- **About 200 visits per door per day.** At 5x, the battery lasts more than 100 days but the offline buffer lasts only 9 days.
- **Key holders can run a CLI and take a phone call at install.** If not, the install check needs a second visit.

### 3.10 First failure
The host's guest Wi-Fi changes its password or re-prompts a captive portal mid-trial. The 12 h alarm fires, and the data stays safe in the ring, so the report later shows no false gaps. The fix is a key holder bringing a hotspot, which is a site visit. The second most likely failure is alignment drift on the steel door as the gasket stiffens in the cold. The clamp keeps the magnet on, but marginal switching shows up as `missed_edge` or suspect hours, ideally within the first 3 daily pulls.
