# Revision: Team Pantry, round 01

## 1. Responses

### Red team
| Finding | Verdict | Reason and change |
|---|---|---|
| Biggest risk: rescue scan on each door close is a live door signal | **Accept** | Right, and it fails a hard constraint. Rescue scans now run only on the hourly timer, at a fixed minute, as passive scans. The status page shows check-ins only as a 6-hour window. Install confirmation moves to the LED and the day-1 decrypt, not the status page. |
| 1. Re-provisioning either can't happen or is an open door | **Accept** | Release-mode flash encryption with no OTA means USB can't change the key list. We keep the lock and make the board the unit of re-keying: removal means swapping in a freshly provisioned board. The device takes no key list over USB. |
| 2. All backups in one envelope | **Accept** | Each holder keeps their own printed backup in their own place. The paper holds only the passphrase-protected key file, as a QR. |
| 3. Token removal is server trust (gap C) | **Accept** | The token stays as a same-day convenience, but it is not counted as removal. Removal is a board swap within 7 days. The Cloudflare account gets two owners with 2FA, and neither is the only tech person. |
| 4. Stuck-closed reads as a true zero (gap A) | **Accept** | This is the worst failure for us, because it feeds wrong zeros straight into the restock decision. Hourly health rows now carry the raw pin level and `hours_since_open`, and the report hatches long daytime silences as "sensor suspect". |
| 5. Identity-key entropy and storage | **Accept** | The key is generated after first boot with the radio on, stored in encrypted NVS, and enrolled with a one-time code. Console off. |
| Gap B: forged ack deletes data; ack clock unbounded | **Accept** | An ack now only moves the upload pointer. Only ring overwrite deletes records. Ack time is applied only within 10 minutes per 6 hours, and anything larger is logged as `clock_reject`. |
| Unchecked: rescue retry budget doesn't match | **Accept** | Fixed by the timer change. Rescue mode now has its own line in the power table. |

### Hardened
| Finding | Verdict | Reason and change |
|---|---|---|
| Biggest risk / 1: nobody looks at counts until day 30; liveness covers only the MCU | **Accept** | Reports now run on days 1, 3, and 7, then weekly. A manual tally happens on day 2. Health rows add wake count and raw level, plus the suspect flag above. |
| 2. Rescue mode missing from the power budget and tied to doors | **Accept** | See red team. It costs 0.9 mAh/day and is now in the table. |
| 3. No chatter limit and no overflow guard | **Accept the limit, reject restoring the threshold upload** | A firmware chatter limit caps how fast events can arrive, so the buffer can't fill between 6-hour uploads, and the threshold trigger would leak busyness again. If the buffer wraps while offline, the device logs `overwritten` with the lost sequence range. |
| 4. No diode on the 5 V pin; USB with cells in | **Accept** | Added a 1N5817 in series. The slide switch is gone: cells go in at the bench, and the field has no USB use left. |
| 5. Batch key across deep sleep; torn writes | **Accept** | The data key and nonce counter live in LP memory, which survives deep sleep and is wiped at power-off. Records are framed as length, sequence number, AEAD, then a commit byte. Boot scans to the last commit and logs `torn`. |
| Unchecked: "300 days at 10x sleep" is wrong | **Accept** | Recomputed below. |
| Unchecked: -20 °C is too warm; the design point is -29 °C | **Accept** | Added a -29 °C line at 0.3 derating. |
| Unchecked: 3.9 V swap threshold; status page voltage contradicts metadata | **Accept** | Voltage is read under TX load, and the threshold is treated as a guess to calibrate. Battery voltage is sent unsealed and rounded to 0.1 V. That is stated now. |
| Unchecked: coating doesn't protect contacts | **Accept** | The lid now opens only indoors (bench, swap). |

### Scrappy
| Finding | Verdict | Reason and change |
|---|---|---|
| Biggest risk: no early look at counts; no tally date | **Accept** | Same fix as Hardened's finding 1. |
| 1. Rescue scan on door close | **Accept** | As above. We use an hourly scan, not every 15 minutes, because hourly is enough for a minder holding a hotspot. |
| 2. USB re-provision blocked by flash encryption | **Accept, option (b)** | We keep flash encryption and swap the board. This also covers a firmware bug found in week one. |
| 3. Opening the box outdoors on install day | **Accept** | Power-on and lid close happen at the bench. Install mode is armed by the first door change, not by power-on. |
| 4. Steel shrinks the gap; the fallback breaks the budget | **Accept** | Add a 10 mm plastic spacer under the contact and the magnet, and bench-test on a steel plate before ordering. The SM-226L stays only as a tested contingency. |
| 5. Box on cold VHB only | **Accept, modified** | Two rubber pot magnets carry the box, plus VHB pads as a second hold. Red team showed magnets alone invite casual lifting. Both together take a deliberate pull. |

## 2. Stolen ideas
- **Scrappy:** batch key in RTC/LP memory, drawn after an upload; Schottky on 5 V; board swap as re-provisioning; magnet-mounted box; hand warmer on the surface; three long blinks on first upload.
- **Hardened:** hourly liveness heartbeat (amends 0001); sequence number and commit byte in the envelope, with sequence breaks shown as gaps; hourly raw-level check (`missed_edge`); steel spacer; offline scan driven by the heartbeat, not doors.
- **Red team:** ack moves a pointer and never deletes; bounded ack clock; two Cloudflare owners; own-place backups.

## 3. The design now

### 3.1 The design in three sentences
Each door gets its own identical counter: a wired alarm contact on spacers, feeding a clear-lid IP68 box with an ESP32-C6 board and three retail AA lithium cells, closed at the bench and held to the fridge by magnets plus tape with no drilling. The board sleeps until the door moves, seals each event to the named key holders, writes an hourly heartbeat, and uploads every 6 hours over the host's guest Wi-Fi, falling back to a volunteer's phone hotspot on a fixed hourly timer. A key holder runs one command on days 1, 3, and 7 and then weekly, which turns sealed batches into an hour-by-day chart for the volunteer chat, with gaps and "sensor suspect" hours hatched.

### 3.2 Parts (per door; the trial builds two plus one spare board)
| Part | Price | Source |
|---|---|---|
| Seeed XIAO ESP32C6 | $5.20 | seeedstudio.com (checked, 10+ tier; single-unit price unverified) |
| Seco-Larm SM-200Q/W surface contact, N.C., wired | $2.71-5.95 | JMAC / FPC Security (checked) |
| 3x Energizer L91 lithium AA | $9.00 | Walmart/Target (checked, ~$12/4) |
| Hammond 1554C2GYCL, clear lid, IP68 | ~$17 | Farnell (Scrappy checked ~¥2,600; US price unverified) |
| 3xAA holder with leads, no switch | $2.00 | Adafruit/Amazon (unverified) |
| PG7 nylon gland | $1.00 | Amazon (unverified) |
| Indicating silica gel, 10 g | $0.50 | Amazon (unverified) |
| 2 rubber-coated pot magnets (box mount) | $3.00 | Amazon/hardware (unverified) |
| VHB 5952 pads, 4 adhesive clips, 2 scrap 10 mm plastic spacers | $3.00 | hardware store (unverified) |
| 1N5817, 1 MΩ pull-up, 2x 1 MΩ divider, conformal coat | $0.60 | Digi-Key (unverified) |
| Vinyl label outside, card inside lid | $1.50 | print shop (unverified) |
| **Total** | **$45.51-48.75** | Under $50. Install kit (shared): hand warmer, alcohol wipes, paper template. Server $0 (free tier, unverified). |

### 3.3 Power (`skills/enclosure-and-power`; 50 visits/day = 100 events; 3000 mAh, Energizer's 200 mA figure)
| State | mA | h/day | mAh/day |
|---|---|---|---|
| Deep sleep, board (Seeed wiki 15 µA at 3.8 V; measure via 5 V pin) | 0.015 | 24 | 0.36 |
| Regulator and diode leakage (unverified, measure) | 0.020 | 24 | 0.48 |
| Contact pull-up, door shut | 0.0033 | 24 | 0.08 |
| Battery divider, 2 MΩ | 0.0025 | 24 | 0.06 |
| Door wake, debounce, seal, commit (100 x 0.15 s) | 25 | 0.0042 | 0.10 |
| Hourly heartbeat (24 x 0.2 s) | 25 | 0.0013 | 0.03 |
| Upload: Wi-Fi, TLS, sign, verify ack (4 x 8 s) | 100 | 0.0089 | 0.89 |
| Retry allowance: one failed 15 s attempt | 100 | 0.0042 | 0.42 |
| **Total, normal** | | | **2.42** |
| Rescue mode instead of the last two rows: 4 failed 15 s guest attempts + 24 x 1.5 s passive scans at 90 mA | | | 1.67 + 0.90 |
| **Total, rescue mode** | | | **3.68** |

Life at freezing = 3000 x 0.5 / 2.42 / 1.3 = **476 days**. In rescue mode for the whole time it is 313 days. At -29 °C with 0.3 derating, it is 286 days. If sleep draw is 10x higher, it is 204 days. A worst-case chatter storm, capped by the firmware limit, adds about 0.75 mAh/day. Encryption and signing times are guesses until measured on the C6. The trial needs 30 days. The "change once a year" plan is a goal to check against field data, not a trial claim.

### 3.4 Behavior
- **Wake (0001):** a pin interrupt, then 50 ms settle and 500 ms lockout. Only state changes are emitted.
- **Chatter limit:** if there are more than 30 wakes in 2 minutes, the device writes a `chatter` row and disables pin wake until the next heartbeat. The report shows that hour as a gap.
- **Heartbeat:** every hour, on the RTC timer. A sealed health row holds `battery_mv` (under load at the last TX), `boot`, the raw pin level, `wakes`, `hours_since_open`, `torn`, `overwritten` range, `clock_reject`, and drift. If the raw level disagrees with the last state, it logs `missed_edge` and never invents an event.
- **Write path:** `[length][sequence][ChaCha20-Poly1305 record][commit byte]`. Boot scans to the last commit and logs any torn tail.
- **Seal:** libsodium. The data key is drawn right after each ack while Wi-Fi is up, or at cold boot after the radio is started. It is wrapped with `crypto_box_seal` to each holder and the header is written to flash. The key and nonce counter live in LP memory, which is wiped at power-off.
- **Upload:** a fixed 6-hour schedule, padded to a fixed size, with no threshold trigger. Batches are signed with Ed25519. A signed ack naming the device and batch moves only the upload pointer. Records leave flash only through ring overwrite, and the ring holds about 100 days. The ack time is applied only if it is within 10 minutes of the device's own time.
- **Offline:** after 24 hours with no ack, each hourly heartbeat at a fixed minute does a passive scan for the rescue SSID. It connects only if the SSID is seen, and uploads everything not yet acked. It never acts on door activity.
- **Install mode:** armed once per provisioning by an NVS flag, and started by the first door change. For 10 minutes or 20 events, the LED blinks on each change. Then the device makes one upload and gives three long blinks on success. After that the flag clears and the LED stays dark for good, even after a reboot or a battery swap.
- **Server:** a Cloudflare Worker with R2. It checks the device signature, stores the sealed batch, and signs the ack. It keeps unsealed only the device ID, batch ID, check-in time, and battery to 0.1 V. An R2 rule deletes batches after 90 days. The account has two owners, both with 2FA.
- **To a person:**
  - **Minder email:** if 4 windows are missed, the minder gets: "Fridge counter has missed check-ins since the Tue noon window."
  - **Status page:** shows each device's last 6-hour window and battery voltage, never a minute-level time.
  - **Reports:** `latchwing report` decrypts in memory and pairs events per `time-bucketing`. It writes a PNG and CSV by hour, day of week, and week. It hatches gaps (no heartbeat, a sequence break, chatter) and "sensor suspect" hours (more than 6 daytime hours with no open on a door whose daytime median is at least 1 an hour, or any open longer than 1 hour). Raw rows never touch disk unless asked with `--raw`. A key holder runs it on days 1, 3, 7, 14, 21, and 28, and posts the PNG to the volunteer chat and the public page.
- **The decision it changes:** when the daily restock drop happens, and whether the busiest days need a second drop. Example: "Saturdays empty by 2 p.m., so drop at 11 a.m. and add a 4 p.m. run." The fridge-versus-cabinet split decides how much shelf space goes to cold versus dry goods.

### 3.5 Install and upkeep
**Bench (tech key holder, about 1 hour per device):** solder 5 joints, coat the board, and run `latchwing provision`. Bench-test the contact on a steel plate with spacers. Do a 24-hour freezer test and 20 tallied opens. Insert the cells, check that the desiccant is blue, and close the lid. Pack the door magnet in a separate envelope so install mode can't start in transit.

**Site (any volunteer, about 20 minutes, from a one-page sheet):**
1. Wipe the frame and the door edge with alcohol. Hold the hand warmer on each spot for 5 minutes, through a folded cloth.
2. Tape the paper template on. Stick the spacers, then the contact on the frame and the magnet on the door at the marks, with the gap under 1/2 inch. Press each one for 30 seconds.
3. Set the box on the fridge side wall with its magnets and VHB pads, gland down, above the snow line and clear of the door swing. Clip the cable flat.
4. Open and close the door 10 times: 20 blinks, then 3 long blinks. No blinks means re-check the alignment. Blinks but no long ones means tell the key holder that the Wi-Fi didn't connect.
5. Send the key holder a photo. On day 1 they decrypt and confirm the 20 events.

**Upkeep:**
- **Day 2:** a volunteer does a 2-hour manual tally. The key holder compares it with the day-3 report.
- **Alert email**, following the card:
  1. Ask the host whether the Wi-Fi changed.
  2. Check that the box and cable are intact.
  3. Turn on a phone hotspot with the card's name and password, and leave it on near the fridge for at least an hour.
- **Report shows suspect hours or long opens:** a volunteer re-sticks the parts, using the template.
- **No routine work during the trial.** After the trial, when the status page shows low voltage under load or the desiccant turns pink through the lid, take the box indoors and swap the cells and desiccant. This needs one screwdriver.

### 3.6 Data access (`skills/access-control`)
- **Who:** 2 or 3 named key holders (for example, the coordinator, the tech volunteer, and a board member). Names stay in pantry records, not in this repo.
- **How they get the data:** `latchwing keygen` creates an X25519 key protected by a passphrase. Each holder prints that protected file as a QR code and stores it in their own place, never together with another holder's. Downloads need a per-holder token. The batches stay sealed either way.
- **Removal:** revoke the token the same day, as a courtesy and not as the control. Within 7 days, the remaining holders provision the spare board with the new key list. A volunteer lifts the box off, opens it indoors, and moves the contact wires and battery plug to the new board, which takes about 10 minutes. The old board powers up once near the rescue hotspot to upload what is left, and then its identity key is revoked on the server. Cloudflare access is reviewed at the same time. The removed holder keeps anything they already read, as 0006 accepts.
- **Lost laptop:** revoke the token, restore the key from that holder's own paper backup, and keep reading. The key file is useless without the passphrase.
- **Provisioning:**
  1. `latchwing provision` flashes signed firmware (Secure Boot v2) with the holder public keys, the ack public key, door ID, guest Wi-Fi and rescue logins, and a one-time enrollment code.
  2. It turns on flash encryption in release mode and NVS encryption. It burns eFuses to disable JTAG and USB-JTAG, sets secure download mode, and turns the console off. There is no OTA, no BLE, and no access point.
  3. On first boot the app starts the radio and only then generates the Ed25519 identity key into encrypted NVS. It enrolls over the rescue hotspot using the code, and the server burns the code.
  4. A flash-dump check on the spare board must show no readable events or keys.
- **Chip:** the ESP32-C6 has a hardware RNG, Secure Boot v2, flash and NVS encryption, eFuse debug lock, and retained LP memory. Fault injection (AR2023-007) is lab-grade and exposes only the identity key and Wi-Fi logins.
- **Lost device:** revoke its identity key on the server. It shows up as a gap.

### 3.7 Decisions
- **0001 Keep, amended.** Door events stay interrupt-only. We add an hourly timer heartbeat for liveness, which never creates a door event, and a firmware chatter limit.
- **0002 Hold.** The four fields are unchanged. The sequence number and commit byte go in the envelope, not the record. New health fields are listed in 3.4.
- **0003 Change.**
  - The fixed 6-hour padded schedule replaces "schedule or threshold". The chatter limit bounds the event rate, so the threshold trigger isn't needed.
  - An ack moves an upload pointer and never deletes. Only ring overwrite deletes.
  - Ack time is bounded.
  - A rescue hotspot is a second transport, scanned only on the hourly timer.
- **0004 Hold.** Gaps come from missing heartbeats, sequence breaks, or chatter. "Sensor suspect" hours are hatched like gaps, never shown as zero.
- **0005 Change.** "Serviceable enclosure" replaces "sealed enclosure": IP68 with a clear lid and 4 screws, retail AA cells, and a coated board, opened only indoors. Mains stays rejected.
- **0006 Keep, clarified.** Removal is a board swap within 7 days, and that is the provisioning step 0006 names. Tokens are a convenience only. Each holder keeps their own backup. The report decrypts in memory. The ack key is bounded by the pointer rule and the clock bound.

### 3.8 Open questions
1. **Dead versus quiet:** an hourly sealed heartbeat, written with or without Wi-Fi. An hour with a heartbeat, no sequence break, and no suspect flag is a true zero. Anything else is hatched. The server's missed-window email tells offline apart from dead.
2. **Mains:** still rejected. The outlet is the host's, usually behind the fridge, and a cord to the door frame is the easiest thing to yank. The battery covers more than 9 times the trial length even at -29 °C.
3. **One device per door.** No cable of unknown length, one set of instructions, and independent failure.
4. **Pulling and looks:** magnets plus VHB take a deliberate pull, and the cable is clipped flat. A torn contact reads as a long open, and a lifted box shows as missed windows. Either way, a gap. The clear lid shows only cells and a small board. The label reads: "Door counter. Counts how many times this door opens so volunteers know when to restock. No camera. No microphone. It cannot tell who you are. Totals: [QR]. Questions: [contact]." No light after install.

### 3.9 Assumptions
| Assumed | If wrong |
|---|---|
| Guest Wi-Fi reaches both doors | Data arrives only through the rescue hotspot, which fails "no site visit". v2 needs LoRa. |
| No outlet needed | No change. |
| Steel fridge with a flat frame spot; cabinet is wood | Plastic fridge: drop the spacers, and the box goes on VHB with a strap. Wood cabinet: the box goes on VHB plus 2 screws if the host allows. |
| Host allows magnets and adhesive, no drilling | If adhesive is refused, the contact needs a clamp bracket, which means a redesign. |
| Fridge and cabinet are near each other | Irrelevant. |
| 2 or 3 holders, with one tech-capable | With one tech person, re-keying and swaps depend on one person. |
| About 50 visits a day; coldest week -29 °C | At 10x traffic, life is still more than 300 days. L91 cells are rated to -40 °C, and TX sag in the cold is unmeasured. |

### 3.10 First failure
The contact or door magnet is bonded on cold steel in winter, and it creeps. Counts drift, or a stray magnet holds the reed closed. The day-3 and day-7 reports catch it as long opens or "sensor suspect" hours, and the day-2 tally confirms it, so it surfaces in the first week, not on day 30. The fix is a re-stick visit, which counts against "no site visit". The second most likely failure is that the host changes the Wi-Fi. Nothing is lost: the minder's hotspot recovers the data within an hour.
