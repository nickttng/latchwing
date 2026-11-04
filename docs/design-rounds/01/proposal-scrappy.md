# Proposal: Team Scrappy, round 01

## 1. The design in three sentences
Each door gets its own copy of the same build: a $5 Seeed XIAO ESP32-C3, a stock wired alarm reed contact (MC-38), and three lithium AA cells, all in an off-the-shelf IP65 junction box. The chip sleeps until the door changes state, logs one sealed event, and every 6 hours wakes on a timer to write a health row and upload sealed batches over the site's guest Wi-Fi to a free-tier Cloudflare Worker that can store batches but cannot read them. Key holders download the batches with a small script, open them on their own laptops, and publish only hourly counts with gaps marked.

## 2. Parts (per device)
| Part | Price | Source |
|---|---|---|
| Seeed XIAO ESP32-C3 (antenna included) | $4.99 | seeedstudio.com, checked 2026-11-03 |
| MC-38 wired reed contact + magnet, normally open | ~$3 (5-packs ~$10–15) | Amazon/eBay, **unverified**, pack price varies |
| 3x Energizer Ultimate Lithium AA (L91) | ~$9 ($23.98 for 8) | Walmart listing, not dated, **unverified today** |
| IP65 ABS box, 100x68x50 mm | $10.99 | Newegg Business listing |
| 3xAA plain holder with leads | ~$2 | **unverified** |
| PG7 cable gland | ~$0.70 | 10-packs on Amazon, **unverified** |
| Schottky diode (1N5817), 1 MΩ pull-up, 2x 1 MΩ divider | ~$0.50 | parts bin or kit, **unverified** |
| Silica gel pack, 5 g | ~$0.30 | **unverified** |
| 2 rubber-coated neodymium cup magnets (box mount) | ~$3 | **unverified** |
| 3M VHB 5952 strip (sensor + door magnet) | ~$1 share of a roll | **unverified** |
| Printed notice label | ~$0.50 | |
| **Total** | **~$36** | Under $50. Server: $0 (free tier, **unverified** limits). |

No custom board. Everything is solder-four-wires or screw-terminal work. The only part with a long lead time risk is the MC-38 if bought from overseas sellers; buy a US-stocked pack.

## 3. Power
Cells: 3x L91 in series (4.5–5.4 V) through the Schottky into the XIAO 5V pin (Seeed's wiki allows 5V input through a diode), onboard regulator to 3.3 V. Fridge door assumed 100 opens/day = 200 events.

| State | Current (mA) | Time per day (h) | mAh/day |
|---|---|---|---|
| Deep sleep (Seeed spec 44 µA) | 0.044 | 24 | 1.06 |
| Regulator + diode leakage (**unverified**, measure) | 0.020 | 24 | 0.48 |
| Reed pull-up, 1 MΩ, door closed | 0.0033 | 24 | 0.08 |
| Battery divider, 2 MΩ | 0.0025 | 24 | 0.06 |
| Wake, debounce, seal, log: 200 x 0.25 s | 25 | 0.0139 | 0.35 |
| Upload + health row + batch sign: 4 x 10 s | 110 | 0.0111 | 1.22 |
| **Total** | | | **3.25** |

Life = 3000 mAh x 0.5 cold / 3.25 / 1.3 = **~355 days at freezing.** If deep sleep is 10x worse than spec (a bad clone board), life is still ~90 days. The 30-day trial does not hinge on power; measure sleep current on the bench anyway and log `battery_mv` in health rows.

## 4. Behavior
- **Wake:** reed on GPIO3 (D1). Before sleep, firmware arms GPIO wake for the level opposite the current state (the C3 wakes on level, so this gives wake-on-change). On wake: 50 ms settle, re-read, emit only a state change, 500 ms lockout. Per `event-driven-wake`.
- **Seal:** per `access-control`. The batch data key is drawn from the hardware RNG right after an upload, while Wi-Fi is on (the C3's RNG is only fully random with the RF or bootloader entropy source enabled). It lives in RTC slow memory, which survives deep sleep and is lost on power-off. A cold boot starts a new batch.
- **Store:** sealed JSON-lines records in a 1.5 MB flash ring partition on the 4 MB chip (sizing rule: 200 events x ~90 bytes x 30 days x 2 = ~1.1 MB needed).
- **Upload:** every 6 hours on a fixed timer only, not on activity, so upload timing does not track busyness. A 50 percent-full trigger stays as an overflow guard that the trial should never reach. The batch is signed with the device identity key; the Worker checks it, stores it in R2, and returns a signed ack with device ID, batch ID, and time. The device advances its read pointer only on a valid ack and syncs its clock from it. No ack: keep the batch and retry next window.
- **Health row:** at every 6-hour wake, uploaded or not: `battery_mv`, `boot` count, measured clock drift, Wi-Fi RSSI, and `bounce_rejects` (debounce rejections, a count only).
- **To a person:** the Worker emails the key holders if a device has not uploaded in 24 hours (a liveness signal, no door data). Once a week a key holder runs `latchwing pull` (downloads batches with their token) and `latchwing report` (opens batches with their private key, pairs opens and closes, buckets by hour, day, and week, marks gaps). They post the CSV and a one-page chart to the pantry's shared folder.

## 5. Install and upkeep
A key holder provisions and tests both devices at home (freezer 24 h, then 20 tallied opens). The volunteer gets a box, a bag of parts, and a one-page card:
1. Check the guest Wi-Fi on a phone standing at the fridge. Two bars or better, or stop and tell the key holder.
2. Wipe the frame and door edge with the alcohol wipe. If below 50 °F, hold the hand warmer on each spot for 2 minutes.
3. Stick the sensor on the frame at the latch edge and the magnet on the door, lined up with the printed marks, gap under 10 mm.
4. Put the box on the fridge side wall with its magnets, gland facing down, at least 15 cm from the sensor. Cabinet: two screws.
5. Insert the batteries. The LED blinks once per door change for 10 minutes. Open and close 10 times and count 20 blinks. Three long blinks means the first upload worked.
6. Close the lid, stick on the notice, send the key holder a photo.
Upkeep: nothing for 30 days. If the liveness email fires or the weekly report shows chatter, a volunteer re-checks alignment. Separately, a volunteer does two 2-hour manual tallies in the trial for the 5 percent check.

## 6. Data access
- **Who reads raw events:** two or three named key holders, names kept outside the repo. Batches are sealed to their X25519 public keys with libsodium; records use ChaCha20-Poly1305 with a counter nonce.
- **How they get them:** each holder has a personal download token for the Worker. The server, R2, the network, and a USB pull hold only sealed blobs. R2 lifecycle rule deletes blobs after 90 days, and holders delete local raw copies on the same schedule.
- **Removing someone:** delete their download token on the Worker right away (they lose access to new blobs). Their public key is dropped at the next provisioning: end of trial, or sooner by swapping in a freshly provisioned board (two screw terminals). They keep what they already read.
- **Provisioning (key holder's laptop, ESP-IDF):** build firmware with holder public keys and the ack public key compiled in; put Wi-Fi guest credentials in encrypted NVS. On first boot the device generates its Ed25519 identity key with the RNG and prints the public key once; the holder enrolls it on the Worker. Then burn eFuses: Secure Boot v2, flash encryption in release mode, USB-JTAG and pad JTAG disabled, secure download mode. No OTA in v1. The ack signing key is a Worker secret; its public half is in the firmware.
- **The chip needs:** hardware RNG, secure boot, flash encryption, and debug lock. The ESP32-C3 has all four; no secure element.

## 7. Decisions
- **0001 keep.** Reed switch is the cheapest sensor, and the stock alarm contact is already sealed. The C3's level-wake needs the opposite-level trick above.
- **0002 keep.** Four fields, health rows separate. The XIAO has no 32 kHz crystal, so the clock runs on the internal RC. Drift is logged at each 6-hour sync and corrected downstream; batches past 5 minutes are flagged.
- **0003 keep, with one change:** uploads run on the 6-hour timer only. The threshold trigger is just an overflow guard. Without the radio, a dead device would go unnoticed until day 30, and the clock could only be synced from a signed ack at the end. That breaks the trial's "no site visit" goal and our first principle.
- **0004 keep.** The reporting step is a 100-line script, not a dashboard. Without it, the privacy rule and the gap rule have no home.
- **0005 keep.** IP65 costs $11 off the shelf, so there is nothing to save by cutting it. Batteries, not mains: see Q2.
- **0006 keep, with two notes.** "RAM only" means RTC slow memory across deep sleep, wiped on power loss. Removal also revokes the holder's server token right away.

## 8. Open questions
1. **Dead vs quiet:** the device writes a health row every 6 hours whether or not the door moves. An hour bucket counts as data only if health rows exist on both sides within 6 hours; otherwise it is a gap. A `boot` row marks resets. The Worker's 24-hour silence email catches a dead device within a day.
2. **Mains:** still rejected for v1. We don't know who controls the outlet, a cord is one more thing to unplug or trip over, and outdoor wiring needs a weather-rated adapter. The budget gives roughly 12x the trial length on three AAs, so mains solves nothing the trial hits.
3. **One device per door.** Distance is unknown, a cable between doors is a snag and install risk, and one failure then costs one door, not both. Two identical builds also give us a built-in comparison.
4. **Pulling and looks:** the box is held on by magnets, so a yank pops it off without damaging the fridge. A torn sensor wire reads as a stuck open, which the report flags (open over 1 hour = suspect, bucket shown as a gap). A stolen box exposes no sealed data. Looks: plain white box, antenna inside, LED off after install, and a large-print notice: "Door counter. Counts openings only. No camera, no microphone, no tracking. Questions: [pantry contact]."

## 9. Assumptions
- **Guest Wi-Fi reaches the fridge.** If wrong, data still buffers 30+ days and a key holder does one USB-cable pull at the end of the trial through the same `send()` interface. We lose early data and liveness, which is the main loss.
- **No outlet needed.** If one turns out to be free and controlled by the pantry, nothing changes for v1.
- **Fridge skin is steel; cabinet is wood.** If the fridge is plastic or aluminum, the box goes on VHB too. A steel door may shorten the reed's switching gap, so the install card's 10 mm limit gets tested on a steel offcut first.
- **The doors are far enough apart that one cable is awkward.** If they sit side by side, one device per door is still fine; it just costs an extra $36.
- **Key holders:** two people with laptops willing to run a script weekly. If only one, the design fails 0006's two-holder minimum and we wait.
- **About 100 fridge opens a day.** At 1,000, the 1.5 MB ring holds only ~8 days offline, though 6-hour uploads empty it long before then. Power drops to ~180 days.

## 10. First failure
Sensor alignment on the fridge door. Cold-applied tape creeps, the door sags, or someone knocks the magnet, so the gap drifts past the switching point and the door chatters or misses closes. Counts drift past 5 percent before power, Wi-Fi, or the enclosure give out. We'll see it as rising `bounce_rejects` and unpaired events in the weekly report. The fix is a volunteer re-sticking the parts, which is a site visit.
