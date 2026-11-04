# Proposal: Team Pantry, round 01

## 1. The design in three sentences
Each door gets its own identical counter: a wired alarm-style door contact feeding a clear-lid IP66 box that holds an ESP32-C6 board and three retail AA lithium cells, stuck to the fridge or cabinet with no drilling. The board sleeps until the door moves, seals each event to the named key holders, logs a heartbeat every 6 hours, and uploads on that schedule over the host's guest Wi-Fi. Each month a key holder runs one command that turns the sealed batches into an hour-by-day chart for the volunteer chat, with gaps marked, and the batteries last well over a year.

## 2. Parts (per door; the trial builds two)
| Part | Price | Source |
|---|---|---|
| Seeed XIAO ESP32C6 board | $5.20 | seeedstudio.com (checked, 10+ tier) |
| Seco-Larm SM-200Q/W surface contact, N.C., 3/4" gap, wired | $2.71-5.95 | JMAC / FPC Security (checked) |
| Energizer Ultimate Lithium AA (L91), 3 of a 4-pack | $9.00 (pack $12) | Walmart/Target (checked, ~$12/4) |
| Hammond 1554-series IP66 polycarbonate box, clear lid (1554C size) | $13.04-18.23 | Solutions Direct / Digi-Key (checked for smoked lid; exact clear-lid SKU unverified) |
| 3xAA battery holder with leads and switch | $2.00 | Amazon/Adafruit (unverified) |
| PG7 cable gland, nylon | $1.00 | Amazon (unverified) |
| Indicating silica gel pack, 10 g | $0.50 | Amazon (unverified) |
| 3M VHB tape strip and 4 adhesive cable clips | $3.00 | hardware store (unverified) |
| 1 MΩ pull-up resistor, conformal coat, heat-shrink | $0.50 | Digi-Key (unverified) |
| Printed weatherproof vinyl label (outside) and paper card (inside lid) | $1.50 | local print shop (unverified) |
| **Total** | **$38.45-46.88** | Under $50. Server costs $0 (free tier, unverified). |

## 3. Power (per `skills/enclosure-and-power`; 50 visits/day = 100 events)
| State | mA | h/day | mAh/day |
|---|---|---|---|
| Deep sleep, board level (vendor forum figure, unverified, to be measured) | 0.015 | 24 | 0.36 |
| Contact pull-up while door is shut (3.3 V / 1 MΩ) | 0.0033 | 24 | 0.08 |
| Wake, debounce, seal, and log (100 x 0.15 s at 25 mA) | 25 | 0.0042 | 0.10 |
| Heartbeat and upload: Wi-Fi, TLS, sign, verify ack (4 x 8 s at 100 mA) | 100 | 0.0089 | 0.89 |
| Retry allowance: one failed 15 s attempt per day | 100 | 0.0042 | 0.42 |
| **Total** | | | **1.85** |

Life = 3500 mAh x 0.5 (freezing) / 1.85 / 1.3 = **728 days at freezing.** Assumptions: three L91 cells in series feed the board's 5 V pin through its onboard regulator, so cell mAh equals board mAh. Encryption and signing time are guesses until measured on the C6. The budget is planned for a year between changes, not just the 30-day trial. If the measured sleep current is ten times higher (0.15 mA), life is still about 300 days.

## 4. Behavior
- **Wake:** a pin interrupt on the contact, then 50 ms settle and 500 ms lockout, per 0001. The firmware emits only state changes.
- **Heartbeat:** an RTC timer wakes the board every 6 hours. It appends a health row (`door: device`, `battery_mv`, `boot`, and the heartbeat), then tries an upload. The heartbeat is logged even when Wi-Fi fails, so the sealed log itself shows whether the device was alive.
- **Seal:** this follows `access-control` exactly. The firmware uses ESP-IDF's libsodium. The data key is wrapped with `crypto_box_seal` to each holder, and records use ChaCha20-Poly1305 with a counter nonce. A new batch starts right after each upload, while the radio is on, so the hardware RNG has true entropy.
- **Upload:** a fixed 6-hour schedule only. There is no buffer-threshold trigger, so upload timing says nothing about traffic. Each payload is padded to a fixed size. Batches are signed with the device's Ed25519 key, and the buffer is cleared only on a signed ack carrying the time. If the guest network fails for 24 hours, the device also tries a "rescue hotspot" network, once per door close and at most every 10 minutes.
- **Buffer:** about 1 MB of the 4 MB flash: 200 events x 80 B x 30 days x 2, plus health rows. Uploads are padded; stored data is not.
- **Install mode:** for 10 minutes after power-on, the LED blinks on each detected open and the device uploads immediately. After that the LED is dark for good, so there is never a visible "door is open" signal.
- **Server:** a Cloudflare Worker with R2 storage. It checks the device signature, stores the sealed batch, and returns the signed ack. It cannot read anything. An R2 lifecycle rule deletes batches after 90 days. It keeps only unsealed metadata: device ID, batch ID, and check-in time.
- **To a person:**
  - (a) If a device misses 4 check-ins in a row (24 hours), the server emails the "minder" volunteer: "Fridge counter last checked in Tue 3 p.m." No door data is included.
  - (b) A status page lists each device's last check-in and battery voltage.
  - (c) Monthly, a key holder runs `latchwing report`. It downloads, decrypts in memory, and pairs events per `time-bucketing`. It writes a PNG and CSV of counts by hour of day, day of week, and week, with gaps hatched, and never writes raw rows to disk unless asked with `--raw`. The key holder posts the PNG to the volunteer chat and the public page linked from the label.
  - **The decision it changes:** what time the daily restock drop happens, and whether a second drop is needed on the busiest days. For example, "Saturdays are emptied by 2 p.m., so move the drop to 11 a.m. and add a 4 p.m. run." The fridge-versus-cabinet split decides how much shelf space goes to cold versus dry goods.

## 5. Install and upkeep
**Bench, done by the tech volunteer (key holder), about 1 hour per device:** solder 4 joints (contact, resistor, battery leads), coat the board, and run `latchwing provision` over USB-C. The device ships with batteries in, the switch off, and the lid closed.

**Site, done by any volunteer, about 20 minutes, using a printed one-page sheet:**
1. Warm the adhesive parts in a pocket, then wipe the fridge frame and door with the alcohol pad.
2. Tape the paper template to the frame. Stick the contact on the frame and the magnet on the door at the marks, with the gap under 1/2 inch.
3. Stick the box on the fridge side, away from the door swing and above any snow line. Clip the cable flat so there is no loop to grab.
4. Open the 4 lid screws, flip the switch on, check the desiccant is blue, and close the lid.
5. Open and close the door 5 times and watch for 5 blinks. Then check that the status page on your phone says "checked in just now."

**Upkeep:**
- Nothing routine during the trial.
- If the minder gets an alert email, follow a 3-step card:
  1. Ask the host whether the Wi-Fi changed.
  2. Look at the box: is it on the fridge, and is the cable whole?
  3. Turn on a phone hotspot with the rescue name and password from the card, then open and close the door. The status page should update within minutes.
- Every year, or when the status page shows battery under 3.9 V or pink desiccant through the clear lid: swap the 3 AAs (sold at any drugstore) and the desiccant pack. This needs one screwdriver.

## 6. Data access (per `skills/access-control`)
- **Who:** 2 or 3 named key holders, for example the pantry coordinator, the tech volunteer, and one board member. The list of names lives in the pantry's own records, not in this repo.
- **How they get the data:** each holder runs `latchwing keygen` once. It makes an X25519 key file protected by a passphrase, plus a printed paper backup, which goes into a sealed envelope held by the coordinator. Downloads from the server need a per-holder token. The batches are sealed regardless.
- **Removal, in two layers:**
  1. Revoke the person's server token today, so they can no longer download.
  2. At the next provisioning, re-provision with the new key list so new batches are not sealed to them. This is a 15-minute visit with a laptop and USB-C, combined with a battery or desiccant visit if one is due.
  - They keep anything they already decrypted. That is the accepted trade-off in 0006.
- **Lost laptop:** revoke that holder's token. The other holder still reads everything, and the holder can be restored from their paper backup on a new laptop. The key file is useless without its passphrase.
- **Provisioning (`latchwing provision`):** generates the Ed25519 identity key on the chip; loads the holder public keys, the ack public key, the guest Wi-Fi login, and the rescue-hotspot login; registers the device's public key with the server; and burns the eFuses for Secure Boot v2 (signed firmware only), flash encryption, and JTAG and USB-JTAG disable. It never opens an access point and never enables Bluetooth.
- **The chip:** the ESP32-C6 has a hardware RNG, Secure Boot v2, flash encryption, and eFuse debug lock, which covers everything 0006 requires.
- **Lost device:** revoke its identity key on the server.

## 7. Decisions
- **0001 Keep.** A reed contact with interrupt wake is the simplest thing a volunteer can align, using the paper template.
- **0002 Hold.** The four fields are unchanged. Heartbeat, battery, boot, and drift go in health rows.
- **0003 Change.** Keep store-and-forward and signed-ack clearing.
  - Replace "on a schedule or when the buffer reaches a threshold" with "on a fixed 6-hour schedule, padded." A threshold trigger leaks how busy the door is, and the buffer cannot fill in 6 hours.
  - Add a second, pre-provisioned rescue-hotspot network. That way, when the host's Wi-Fi fails, any volunteer with a phone can recover the data without a laptop.
- **0004 Hold.** Gaps are defined by missing heartbeats (Q1).
- **0005 Change.** Keep IP65 or better, downward glands, desiccant, lithium primary cells, and the written budget.
  - Replace "sealed enclosure" with "serviceable enclosure." The cells are retail AA L91, so a volunteer can buy them anywhere. The lid has 4 screws, the board is coated so opening the lid is harmless, and the clear lid shows the desiccant color without opening it.
  - Mains stays rejected (Q2).
- **0006 Keep, with additions.** Per-holder server tokens give same-day removal from downloads. Paper backups mean a lost laptop does not lose access. The report tool decrypts in memory, so raw rows don't pile up on laptops.

## 8. Open questions
1. **Dead versus quiet:** the device logs a sealed heartbeat every 6 hours, whether or not Wi-Fi works. An hour between two heartbeats with no door events is a true zero. Hours after the last heartbeat and before the next one are a gap, shown hatched. Separately, the server's check-in record drives the 24-hour alert. That way "offline but alive" (the data arrives later) and "dead" (heartbeats never arrive) are told apart.
2. **Mains: still rejected for v1.** The outlet is usually behind the fridge and belongs to the host. A second plug on it gets pulled the first time someone needs the socket, and a cord run to the door frame is the easiest thing on the device to yank. At about 2 years of battery life, mains buys nothing a volunteer would notice.
3. **One device per door.** It needs no cable run of unknown length between the fridge and cabinet. The two builds are identical, so there is one set of instructions, and one failure doesn't blind both doors. Each is under $50. The door ID is set at provisioning and printed on the box.
4. **Pulling and appearance:** VHB on clean steel, a short cable clipped flat, and no exposed loop. If the contact is torn off, the log shows an `open` with no `closed`. The report flags any open longer than 1 hour as "check door or sensor," and the pair is dropped from counts. The clear lid shows only batteries and a tiny board, with no lens.
   - The outside label reads: "Door counter. Counts how many times this door opens so volunteers know when to restock. No camera. No microphone. It cannot tell who you are. Monthly totals: [QR to public page]. Questions: [pantry contact]."
   - The card inside the lid repeats it: "Batteries + door counter. Nothing else."

## 9. Assumptions
| Assumed | If wrong |
|---|---|
| Host guest Wi-Fi reaches both doors | Data arrives only by rescue hotspot, about weekly, which fails "no site visit." v2 needs LoRa or cellular. |
| No outlet needed | No change. |
| Steel fridge door with a flat frame spot; cabinet is wood or metal | On thick steel, the 3/4" gap shrinks. Swap to the SM-226L-3Q wide-gap contact (about $18, which brings the total to about $60). |
| Host allows adhesive mounting and no drilling | If they also refuse adhesive, use magnetic strips on the steel; the cabinet then needs another plan. |
| Fridge and cabinet are near each other | Irrelevant: one device per door. |
| Key holders: coordinator, tech volunteer, one board member | If only one tech-capable person exists, they become a single point of failure for provisioning; no one else can re-flash. |
| About 50 visits a day per door, and the coldest week reaches -20 C | At 10x traffic the battery still lasts over a year. L91 cells are rated to -40 C. |

## 10. First failure
The host's guest Wi-Fi changes its password or gets switched off mid-trial, and nobody tells the pantry. The minder gets the 24-hour email and has to use the rescue hotspot, which counts as a site visit. The counts are not lost: they stay buffered and sealed, and arrive late with heartbeats intact, so the report shows no false gaps.
- **Second most likely:** VHB applied below about 10 C bonds weakly and the magnet creeps. Extra or missed counts show up first as long opens in the report. The steps say to warm the parts, and to install on a mild day if one comes.
