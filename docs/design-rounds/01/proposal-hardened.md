# Proposal: Team Hardened, round 01

## 1. The design in three sentences
One identical, self-contained sealed box per door: an ESP32-C3 board on three lithium AA cells, a potted reed sensor on a 1 m cable, and a separate low-drift clock chip. It wakes on every door change and also once an hour on a timer to write a heartbeat health row. That heartbeat is what lets the data tell a dead device from a quiet door. Every record is sealed to the key holders, committed to flash so a torn write is detected rather than silently lost, and uploaded every 2 hours over the host's guest Wi-Fi to a small server that signs acks and raises an alarm when a device goes quiet.

## 2. Parts (per device; prices checked 2026-11-03, US retail, single unit)
| Part | Price | Source |
|---|---|---|
| Seeed XIAO ESP32-C3 (secure boot, flash encryption, hardware RNG) | $5.39 | RobotShop |
| Adafruit PCF8523 RTC breakout, STEMMA QT (#5189), -40 to 85 °C | $4.95 | adafruit.com |
| Littelfuse/Hamlin 59140-1-S-02-A potted reed sensor, -40 °C rated | $2.48 | Future Electronics |
| Hamlin 57140-000 matching actuator magnet | ~$3 (unverified) | DigiKey |
| Hammond 1554C2GY polycarbonate box, IP68 | $18.23 | DigiKey (regional page; unverified for US) |
| 3x Energizer L91 lithium AA (3/4 of a $14 pack) | $10.50 | Walmart/hardware store (prices vary $10-17) |
| 3xAA holder with wire leads | ~$2 (unverified) | DigiKey/Amazon |
| PG7 nylon cable gland, 1 m 2-core jacketed outdoor cable | ~$3 (unverified) | Amazon/hardware store |
| 470 uF solid-polymer cap (-55 °C), 2x 1 MΩ + 1 MΩ resistors, 100 nF caps | ~$2 (unverified) | DigiKey |
| Silica gel pack, 3M VHB pads, 2 tamper-resistant screws, label | ~$3 (unverified) | hardware store |
| **Total** | **~$54.55** | |

This is about $5 over the $50 budget. Per our charter, reliability comes first. The enclosure is the overrun, and it is the part we refuse to cheapen. Two devices cost about $109 for the trial.

## 3. Power (method from `skills/enclosure-and-power`; all currents unverified until measured on the real board)
| State | Current (mA) | Time per day (h) | mAh/day |
|---|---|---|---|
| Deep sleep: board ~60 uA (Seeed quotes 44 uA for the chip, regulator unmeasured) + reed pull-up 3.3 uA + battery divider 1.6 uA + RTC 0.15 uA | 0.065 | 24 | 1.56 |
| Door wake: 400 events/day (200 visits), 150 ms each incl. AEAD seal and flash commit | 25 | 0.0167 | 0.42 |
| Heartbeat wake: 24/day, 200 ms each | 25 | 0.0013 | 0.03 |
| Upload: 12/day, 10 s each (Wi-Fi join, TLS, Ed25519 sign, ack verify) | 110 | 0.0333 | 3.67 |
| Retry and offline-scan allowance: 24 x 1.5 s | 90 | 0.01 | 0.90 |
| **Total** | | | **6.58** |

Life = 3000 mAh (L91 low-drain rating, from retailer listings; check against the Energizer datasheet) x 0.5 cold derating / 6.58 / 1.3 = **175 days at freezing**. Even at 0.3 derating for -20 °F, that is 105 days. Capacity is not the risk. The risk at -20 °F is voltage sag during 350 mA Wi-Fi peaks. We handle it in three ways. The 470 uF polymer cap carries the peaks. Before any upload, firmware reads the battery under load, and below 3.5 V it skips the upload and logs a `low_batt_skip` health row. Brownout reset is enabled, so a sag causes a logged reboot and never a corrupt write.

## 4. Behavior
- **Door wake (0001):** wake on a level change of the reed pin, which has a hardware 1 MΩ/100 nF RC filter. Then a 50 ms settle, a re-read, and one event if the state differs from the last one recorded. The 500 ms lockout is spent in timer sleep, not awake.
- **Heartbeat:** the timer wakes the chip at the top of every hour. It reads `battery_mv`, enclosure `temp_c` (the chip's own sensor), the raw door level, `buffer_pct`, and the record sequence number, and appends a health row. If the raw level differs from the last recorded state, it logs `missed_edge`. It never invents a door event. If the door has read open for more than 30 minutes, it logs `door_open_long` (this can mean a stuck door, a cut cable, or a moved magnet).
- **Write path:** each record is `[length][sequence number][ciphertext+tag]`, followed by a separate commit byte written last. NOR flash can program the commit byte after the body. On boot, the firmware scans to the last committed record, logs `torn_write` for an uncommitted tail, logs a `boot` row with the reset reason, and starts a new batch.
- **Upload:** every 2 hours on a fixed schedule. Batches are padded to fixed size steps so that upload timing and size leak less about how busy the door is. The device signs each batch. It clears a batch only on a signed ack that names the device and batch, and sets its RTC from the time in that ack. It logs drift as a health row.
- **Offline:** after 24 hours without an ack, each heartbeat also does a 1.5 s scan for the host SSID or a `latchwing-pull` phone hotspot. If the buffer wraps, the device logs `overwrote` with the sequence range it lost.
- **To a person:** the server stores the sealed batches. It can't read them. A key holder runs `latchwing pull` weekly. The tool downloads the batches, checks the device signatures, decrypts locally, and builds hourly, daily, and weekly counts. Any hour without a heartbeat is a gap, and so is any break in the sequence numbers. The key holder publishes only those buckets. Separately, the server emails all key holders if a device misses 3 upload windows (6 hours). That alarm uses only upload metadata, so it needs no decryption.

## 5. Install and upkeep
A key holder provisions and closes the box at home. The volunteer gets two labeled, sealed boxes, a printed one-page sheet, and a bag of parts. The volunteer:
1. Picks a day above 40 °F (VHB adhesive won't bond in the cold), or warms the surface with a hand warmer for 10 minutes first.
2. Cleans the fridge's side wall near the latch edge and the door edge with the alcohol wipe. Sticks the sensor on the cabinet and the magnet on the door, using the paper gap gauge (gap under 10 mm when shut, spacer included because the door is steel). Never drills into the fridge, since refrigerant lines can run in the side walls.
3. Screws the box to the shelter post or wall beside the fridge with the 2 tamper-resistant screws, cable gland facing down, at least 18 in. above the ground and out of any drip line. Clips the cable along the frame every 6 in.
4. Opens and closes the door 10 times while counting. The box has no lights. The key holder confirms by phone that the server shows 20 events after the next upload, or asks the volunteer to wait for the hourly heartbeat. The volunteer repeats steps 2-4 on the cabinet.
5. Sticks on the notice: "Door counter. Counts openings only. No camera, no microphone, no names. Questions: [pantry contact]."

Upkeep during the 30 days: none. If the server raises the silence alarm, a volunteer checks whether the box is still there and the magnet is aligned. The cells and desiccant are swapped only at re-provisioning.

## 6. Data access (`skills/access-control`)
- **Who:** 2 or 3 named key holders, whose names are kept off the repo. Each has an X25519 keypair protected by a passphrase, with an offline backup on paper or USB.
- **Sealing:** at each batch start, `bootloader_random_enable()` draws a 256-bit key from the C3 hardware RNG. Without that call, the RNG is not true random when the RF is off. The key is wrapped with libsodium `crypto_box_seal` to each holder's public key, and the header goes to flash. Records use ChaCha20-Poly1305 IETF with a counter nonce. The key stays in RAM only and is wiped at batch close or reboot.
- **How they get it:** each holder has their own download token on the server. Batches stay sealed end to end, and the server, network, and flash only ever hold ciphertext.
- **Removal:** revoke the person's server token immediately, so they get no new batches. At the next re-provisioning (the end of the trial at the latest), flash a list without their public key. They keep anything they already read.
- **Server:** a Cloudflare Worker with D1 storage (free tier, unverified). It checks each device's Ed25519 signature, signs acks with time, and rejects unknown devices. A breached server can forge acks, which would clear data before anyone pulls it. It cannot read data.
- **Provisioning and chip:** the key holder flashes the firmware, signed with Secure Boot V2, plus the config: holder pubkeys, ack pubkey, guest Wi-Fi and hotspot logins, and door ID. The device generates its Ed25519 identity key itself and prints only the public half, which is registered on the server. Then the key holder enables flash encryption in release mode, burns the eFuses that disable JTAG and USB-JTAG, and sets secure download mode. No BLE, no access point, no open console. A test flash dump must show no plaintext.

## 7. Decisions
- **0001: Keep, amended.** Door events stay purely interrupt-driven, but an hourly timer heartbeat is added. Replaces "no polling loop" with "event wake for door state, plus timer wake for liveness only". A heartbeat never creates a door event. Without the heartbeat, a dead device looks the same as a quiet door. That fails our charter.
- **0002: Keep, amended.** The 4-field door record stays. Added: a sequence number and commit byte in the sealed envelope (not in the record), torn-write detection on boot, and more health-row fields (`temp_c`, `buffer_pct`, `missed_edge`, `door_open_long`, `torn_write`, `overwrote`, `low_batt_skip`). Time comes from the external PCF8523, because the C3's internal RC clock drifts badly in the cold. A power cut can now lose at most one uncommitted record, and that loss is itself logged.
- **0003: Hold.** Plus: buffer wrap is logged with the lost sequence range, and uploads are fixed-schedule and padded. The flash buffer is 1.75 MB, about 19k records, which covers 78 days at 250 per day.
- **0004: Keep.** A gap is defined by data: no heartbeat for the hour, or a break in the sequence numbers.
- **0005: Hold.** Mains is still rejected (see question 2).
- **0006: Hold.** Plus: immediate cut-off through the server token, ahead of re-provisioning.

## 8. Open questions
1. **Dead vs. quiet:** the hourly heartbeat. An hour with a heartbeat and no events is a real zero. An hour without a heartbeat is a gap. The server's 6-hour silence alarm tells people within the day.
2. **Mains:** still rejected. The outlet belongs to the host and may already be in use or switched off. A USB brick and cable outdoors are one more thing to pull, freeze, or unplug. The fridge also loses power exactly when we most want data. Our battery margin is about 6x, so mains buys nothing.
3. **One per door.** The fridge-to-cabinet distance is unknown, a long cable is the easiest thing to rip, and separate devices fail independently. If one dies, the other door still counts.
4. **Pulling and appearance:** the box is screwed down, the sensor is flat and potted, and the cable is clipped every 6 in. A cut cable reads open, which triggers `door_open_long`. A missing box stops uploads, which triggers the silence alarm and a gap. No lights, no lens, a plain gray box, and a plain-language notice.

## 9. Assumptions
- **Guest Wi-Fi reaches the fridge at -75 dBm or better.** If wrong, the data arrives only through a phone-hotspot pull at day 30. Nothing is lost (78-day buffer), but the silence alarm becomes useless, and we'd need LoRa in v2.
- **A wooden shelter or wall next to the fridge for screws.** If not, the box goes on the fridge's top or side on VHB with a stainless strap, and it is easier to pull.
- **Fridge door is steel and cabinet door is wood.** If the fridge door is plastic, drop the spacer. If the cabinet is steel, add one.
- **About 200 visits per door per day.** If it is 5x that, battery life is still more than 100 days, and the buffer still holds more than 15 days offline.
- **Two or three willing key holders, comfortable with a command-line tool.** If not, a key holder needs a one-click script, and v1 slips a week.

## 10. First failure
The magnet on the fridge door. It is stuck on with VHB in January on a cold, condensing steel surface, and the door is slammed hundreds of times. It will most likely shift or peel within two weeks, so the count drifts or the door reads permanently open. The heartbeat shows it as `door_open_long` or `missed_edge` rows, and the open/closed pairing breaks. We would know from the data, but only at the next weekly pull, and fixing it takes a site visit.
