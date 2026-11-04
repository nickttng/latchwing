# Verdict, round 01

Judge: no team. I read the brief, the README, decisions 0001–0006, the team charters, the skills the designs cite, all three proposals, all four critiques, and all three revisions. I scored the revisions. Date: 2026-11-03.

## 1. The pick

**Scrappy's revised design is the base, with the fixes and grafts below.** Each door gets its own XIAO ESP32-C3 with a Seco-Larm SM-200Q reed contact, an LED that works only in install mode, three L91 lithium AAs, and a 470 µF cap, in an IP65 box held to the fridge with VHB plus magnets, which costs about $45. Contacts are bonded only to steel that measures 50 °F or warmer. The device sleeps until the door changes, writes a sealed health row every hour, and every 3 hours uploads padded, sealed batches over guest Wi-Fi (with a phone-hotspot fallback) to a Worker that stores them but cannot read or delete them. A key holder checks the data every morning for suspect windows, and once a week the restock coordinator gets hourly counts with gaps shaded, to set restock drop times.

Why this one: it is the cheapest and fastest way to get a real fridge counting before deep winter, and it finds faults the fastest (a daily check, and a live decrypt during install). Its two weak points are the RC clock and a tape bond that needs a warm day. Both can be tested on the bench before installing, and each has a known fallback that costs one part.

## 2. Scorecard

**Privacy and data access: all three pass.** Scrappy passes on two conditions, both fixed below:
- **Install mode re-arms on every power-on.** As written, install mode is "the first 10 minutes after power-on." A cold-night brownout mid-trial would then upload within seconds of each door change for 10 minutes, which is a short live door signal over RF. This is the same kind of failure Red flagged in Pantry's proposal. Fix: graft 1.
- **The scheduled morning decrypt needs an unlocked private key.** An unattended `latchwing pull && latchwing report` has to get the passphrase from somewhere, and the revision never says where. Fix: rule F3.

Hardened and Pantry pass with nothing open. Both arm install mode once per provisioning, run rescue scans on a timer only, keep acks from deleting data, and show only check-in windows.

| Criterion (weight) | Scrappy | Hardened | Pantry |
|---|---|---|---|
| Count accuracy (3) | **4**: branded contact bench-tested on steel when frozen, a warm bond plus a tug test, the closed level learned at install, a live decrypt, and a daily suspect check. Held back by unmeasured RC drift and an ack-reject rule that can lock the clock out (F1). | **4**: the clamped actuator and the PCF8523 remove the two biggest risks to counts and time. But no one has checked that a handle clamp lands within 10 mm of the frame reed, and below 50 °F the frame reed is still on cold tape. | **3**: spacers, the day-2 tally, and reports on days 1, 3 and 7 are good. But step 1 still bonds to cold steel with a hand warmer. 3M's minimum is a 50 °F surface, and both critics said so. This is Pantry's own predicted first failure. |
| Survives unattended (3) | **4**: chatter limit, framed records, cap, `low_batt_skip`, hotspot fallback, an 80-day ring, and a VHB-plus-magnet box. Held back by 8 uploads a day of cold-sag exposure and the install-mode re-arm. | **4**: IP68 polycarbonate, a screwed box, a spare unit, a sacrificial soak, and recovery that doesn't need the server. Held back by more joints that can fail: the I2C RTC, a reed in the lid, and a 1 m cable run to a post. | **3**: no bulk capacitor and no low-battery skip, even though cold TX sag is the shared top power risk. The box's VHB is also bonded cold. Everything else is sound. |
| Volunteer build and install (2) | **4**: six steps, an LED, and a phone confirmation. It needs a day at 50 °F or warmer, and the box is opened on site. | **3**: worm clamp, aluminum backer, screws into a post, a magnet card, and a live CLI call. It needs a post and a handle that takes a clamp. | **4**: the best install flow (closed at the bench, LED, about 20 minutes, box never opened), undercut by the hand-warmer step. |
| Answers a pantry question (2) | **4**: a daily OK-or-rule email and a weekly restock page for the coordinator. | **4**: the same decision, a fixed page, and a QR code to the totals. Reports are weekly after week 1. | **5**: names the decision with a worked example, adds the fridge-versus-cabinet shelf split, and posts reports to the volunteer chat on fixed days. |
| Cost and availability (1) | **5**: about $42–45, all in US stock. | **2**: about $60, $10 over budget. The box price comes from an aggregator snapshot. The trial costs about $181. | **4**: $45.51–48.75, but the US box price and the single-unit C6 price are unverified. |
| Rebuild from repo (1) | **5**: fewest parts, no fabrication. | **3**: more parts, a hand-made backer, a sacrificial unit. | **4**: few parts. |
| **Weighted total** | **50** | **43** | **44** |

**How sensitive the result is.** Scrappy falls to 44 only if both accuracy and survival drop to 3. That would take a failed drift gate and no warm install day together, and each has a fallback (section 7). Hardened would need +2 on survival and +1 on install to tie, so its case does not rest on the numbers in this table. It is the dissent in section 4.

**Numbers I checked:**
- All three power tables add up: Scrappy 4.50 mAh/day and 256 days, Hardened 4.55 and 253, Pantry 2.42 and 476. All three BOM totals add up, and so do Hardened's 19.1k-record and 45-day ring figures.
- None of the power figures decides anything, because every design covers 30 days several times over.
- Scrappy's "about 1.5 percent from 1 minute of edge error" also checks out: at 10 opens an hour, about 0.17 events fall in each 1-minute edge, so about 0.33 of 20, or 1.7 percent. But that 1 minute is an assumption. No one has measured RC drift on this board.

## 3. Grafts

Each one adds no new part type and no new way to fail.

1. **Install mode runs once per provisioning, set by an NVS flag** (Pantry, Hardened). A reboot or a battery swap never starts it again. This is required for the privacy pass.
2. **A sacrificial lock, plus one provisioned spare board** (Hardened; Pantry's spare). These are the same XIAO, about $5 each. Lock the sacrificial board first and run every bench test on it for at least 72 hours, then lock the trial boards. Release-mode flash encryption can't be undone, and this is first-ever firmware. The spare board also covers holder removal within 7 days.
3. **`missed_edge` and `door_open_long` in the health row** (Hardened). These are firmware flags built from the `pin_level` Scrappy already reads.
4. **A median-based suspect rule from week 2** (Hardened). A daytime hour is suspect when it falls in a run of 4 or more zero hours on a door whose median for that weekday and hour is 3 or more. Scrappy's fixed rules stay for week 1. This is report code only.
5. **`temp_c` from the C3's internal sensor in the health row** (Hardened). It explains drift and sag, and it is the cold data v2 needs.
6. **Passive scans for the hotspot SSID** (Pantry). They stay on the hourly timer and put fewer probes on the air.
7. **A tally schedule with the first session in days 1–3** (Hardened, Pantry). See section 8.
8. **Label text with the purpose and a QR code to published totals** (Pantry, Hardened). It reads: "Door counter. Counts how many times this door opens so volunteers know when to restock. No camera, no microphone, no names. Totals: [QR]. Questions: [contact]."

**Fixes the pick needs.** These are mine, not taken from another team.
- **F1, the clock lock-out.** Scrappy refuses any ack that moves the clock more than 5 minutes. Once real RC drift passes 5 minutes, every later ack is refused too, so the clock is never corrected and every window turns suspect. Replace it with this:
  - The device sets its clock from the first signed ack after each cold boot, logged as `clock_set`, and never steps it again.
  - Every ack's offset goes into a health row.
  - The report converts device time to wall time by piecewise-linear interpolation between offsets. Any offset that is more than 2 minutes off the line through its neighbours is flagged `clock_suspect`.
  - After the first set, a forged ack can no longer move the clock at all, and drift can't lock anything out.
- **F2, ring framing.** Put the ring in a partition not marked encrypted, because its contents are already AEAD ciphertext. With flash encryption on, erased space reads back as noise, and the boot scan for the write head breaks. Also correct the claim that the tag is the commit check on the device. After a power cut the data key is gone, so the device can't check tags. Tags are checked in the report (see the commit-byte item in section 5).
- **F3, unattended decrypt.** The morning task may read the passphrase only from the OS keychain of a laptop with full-disk encryption and a screen lock. Otherwise the holder runs it by hand on days 1, 2, 3, 5 and 7, then twice a week. The key file stays passphrase-protected on disk.
- **F4, the 90-day rule on the device.** Acks no longer delete anything, so at low traffic the ring could keep sealed records for more than 90 days. The device erases any ring sector whose newest record is older than 90 days.

**Declined, kept as fallbacks only.**
- **PCF8523:** add it only if the drift gate fails. It costs $4.95, and the total stays under $50.
- **Handle clamp:** use it only if no warm day comes or the host refuses adhesive.
- **Polycarbonate box:** use it only if the ABS box's temperature rating is under -30 °C.
- **Commit byte:** add it only if the power-cut test fails.
- **Bench-closed install:** opening the box on a 50 °F day costs little, and a magnet in transit could start install mode early.
- **Holder-signed local pull:** `dump-sealed` already gives out only sealed frames.
- **Status page:** the trial doesn't need one.

## 4. Dissent

**The strongest case against the pick comes from Hardened, built on Red's biggest risk for Scrappy.** Scrappy bets the trial on two things nobody has measured: an uncompensated RC clock in the cold, and a tape bond that needs a warm day this late in the year. Hardened removes both for about $10 in parts, with a crystal RTC that a breached server can move by no more than 60 s and a clamped magnet. In Hardened's words, "the clock bound in gap B needs a clock the server cannot set," and "a trial that dies in week two teaches nothing."

**Why it doesn't carry the round:**
- Both bets can be tested before install for almost nothing: the drift gate and the steel-surface reading.
- The clock barely touches the count criterion if tallies are matched against raw events (section 8), rather than hour buckets.
- Hardened's own fixes are unverified. No one has checked the clamp geometry, and below 50 °F its frame reed is still on tape.
- What Red asked for is met by F1. After the first set, a forged ack can't move the device clock at all. A server breached from the start could still shift times by one fixed offset after a cold boot. That residual moves timestamps; it reads nothing and deletes nothing.

If the drift gate fails, Hardened is right about the clock, and the PCF8523 goes in.

## 5. Unresolved

| Disagreement | Who | Cheapest test that settles it |
|---|---|---|
| RTC chip, or RC clock plus downstream correction | Hardened and Red vs Scrappy | 48 h on the sacrificial C3 with at least 4 moves between freezer and room, with acks every 3 h from a bench Worker. Gate: under 60 s of leftover error after interpolation at every point. Fail: add the PCF8523. |
| Upload every 3 h or every 6 h (cold-sag exposure vs drift) | Scrappy vs Hardened and Pantry | The same run, recomputed with 6 h anchor spacing. If 6 h also passes the 60 s gate, move to 6 h. |
| Commit byte, or framing with tags checked in the report | Hardened vs Scrappy | Cut power 30 times mid-write in the freezer. Pass if the report recovers every frame except the torn one and the device keeps appending correctly. Fail: add the commit byte. |
| Tape vs a mechanical hold for the door magnet | Hardened vs Scrappy and Pantry | Warm-bond the contact and magnet on a steel offcut, cure for 72 h, freeze, then 500 slams. Measure gap creep, then pull with a luggage scale. In the field, the day 1–3 tally plus the daily suspect check catch any creep. |
| Box on the fridge (VHB plus magnets) or screwed to a post | Scrappy and Pantry vs Hardened | Ask the host (section 7), then do the same cure, freeze and pull test on the box. |
| Sealed or serviceable enclosure (0005) | Pantry vs the rest | It doesn't matter for a 30-day trial. Settle it in v2 with this trial's desiccant and battery data. |
| Traffic: 50, 100 or 200 visits a day | All three assume different numbers | A volunteer does a 1-hour count before install. It sets the tally length, and at more than 1,000 opens a day it means rechecking the ring size. |

**A shared assumption nobody checked: two 2-hour tallies can confirm "within 5 percent."** At about 10 opens an hour, one session holds about 20 opens, so a single miss is already 5 percent. All three teams copied this plan, and it can't tell 0 percent from 5 percent. Section 8 sizes the tally by number of events instead.

The teams also share three other untested assumptions: that guest Wi-Fi reaches the fridge without a captive portal, that cold TX sag through the diode and LDO is survivable, and that the free-tier Worker can send email. Those three go in sections 7 and 8.

## 6. Decision changes

These are proposed replacement texts. I have not edited `docs/decisions/`, because accepting them is the user's call.

- **0001, amended.** Add: "A timer wake every hour writes a health row and never creates a door event. If there are more than 20 wakes in 1 minute, the device writes a `chatter` row and turns pin wake off for 10 minutes. That window is reported as a gap."
- **0002, amended.** Replace the clock bullet with: "The clock is the microcontroller's own RTC. It is set from the first signed acknowledgment after each cold boot and is not stepped after that. Every acknowledgment's offset is logged in a health row, and wall time is reconstructed downstream by interpolating between offsets. Each record's sequence number sits in the sealed envelope, not in the four fields." Keep the four fields.
- **0003, changed.** Replace "A batch is cleared only after a signed server acknowledgment" with: "A signed acknowledgment advances the upload pointer and deletes nothing. Records leave flash only through ring overwrite, which is logged with the lost sequence range, or through the 90-day expiry." Replace "Upload happens in batches on a schedule, or when the buffer reaches a threshold" with: "Uploads run on a fixed 3-hour schedule (6 hours if the drift test allows), padded to size steps. A 50-percent-unsent guard is the only other trigger." Add: "Transports: guest Wi-Fi, a key holder's phone hotspot scanned only on the hourly timer, and a USB dump of sealed frames."
- **0004, amended.** Replace the gap bullet with: "Gaps are shown as gaps, never as zero. A gap is a dead battery, an outage, a missing health row, a sequence break, a chatter lockout, a `clock_suspect` window, or a suspect-sensor window."
- **0005, unchanged.** Mains is still rejected, and the ABS IP65 box meets the decision. The 50 °F surface rule goes in the install card.
- **0006, amended.**
  - Replace "Each batch uses a new random data key, kept only in RAM" with: "...kept only in RAM or RTC memory, which survive deep sleep but not power loss, and drawn while the radio is on."
  - Replace "Removing a key holder takes effect at the next provisioning" with: "Removing a key holder takes effect within 7 days, by swapping in a freshly provisioned board. Revoking a server token is a courtesy, not the control."
  - Add: "The cloud account has two owners with 2FA, and neither is the only technical volunteer. A private key may be unlocked unattended only from the OS keychain of a laptop with full-disk encryption."

## 7. Check first

**Before buying parts:**

1. **Guest Wi-Fi at the exact mount spot**, with no captive portal, and the host agreeing to warn the pantry before changing the password. *Changes more than the pick:* without it, no design meets "no site visit," and round 02 has to be about the transport (LoRa, or a USB-pull logger with a different success test).
2. **Whether the host allows adhesive and magnets on the fridge.** *Changes the pick:* if adhesive is refused, all three mounts fail, and v1 needs a mechanical mount (Hardened's clamp and a post), which means Hardened's mounting approach becomes the base.
3. **Whether the steel will reach 50 °F before the install date.** Read the frame and door skin with the IR thermometer on a cold day first, since fridge heat may keep them warm. Check the forecast too. Chicago highs fall through November (not verified here). *Changes the BOM, not the base:* if no warm day comes by the time the bench gates pass, either slip the start to the next warm spell or graft the handle clamp for the door magnet.
4. **Door and frame material, and a flat spot at the latch edge.** If the fridge is not steel, the box goes on VHB only and the spacers come out. This doesn't change the pick.
5. **Two or three named key holders**, at least one with a laptop that meets F3 and who can take a call at install. *With only one holder, stop:* 0006 needs at least two.
6. **A 1-hour traffic count at each door.** It sizes the tally. It changes the pick only at more than 1,000 opens a day.
7. **Two Cloudflare owners with 2FA, and the email binding tested on each holder's verified address.**
8. **The ABS box's rated low temperature.** If it is under -30 °C it is fine; if not, swap in the Hammond polycarbonate box, which puts the total at about $48–51.

The outlet and the distance between the doors don't matter, because mains stays rejected and each door gets its own device.

## 8. Trial plan

### Bench gates (sacrificial board first, then the trial units)

Any failure here stops the trial before install.

| Gate | Pass |
|---|---|
| 24 h in the freezer, then 20 tallied opens | 20/20 opens and closes, no extra events |
| Drift (section 5) | Under 60 s leftover error. A fail adds the PCF8523; then retest. |
| Power cut, 30 times in the freezer | Everything recovered except the torn frame, and `torn` logged |
| Sag: lowest 3V3 during upload, fresh and half-used cells, frozen | No brownout on fresh cells; `low_batt_skip` fires before a reset |
| Sleep current through the 5 V path | Budget recomputed, still more than 3x the trial |
| Contact on a steel offcut, frozen, with the spacer used | Switching distance at least 2x the installed gap |
| Security (`skills/access-control`) | Flash dump of the fully provisioned unit shows no usable keys or logins. A batch opens with each holder key and fails with any other. A forged or replayed ack moves nothing. An unknown device is rejected. No console, BLE or AP. Two cold boots give different wrapped keys. Install mode doesn't re-arm after a power cycle. The repo scan finds no secrets. |

### What to measure in the field

- **Counts:** per door, per hour, plus unpaired events, `bounce_rejects` and chatter windows.
- **Completeness:** share of hours with a health row; gap hours by cause; suspect hours; sequence breaks; `overwritten`; `torn`; every ack offset against `temp_c`; `clock_suspect`; `low_batt_skip`; `battery_mv` under load against temperature; RSSI; upload success rate; hotspot uses.
- **Operations:** every site visit, with its cause and who went; time from a fault to a person knowing; install time and every stumble, written down by the volunteer; key-holder minutes per week.
- **Use:** whether the coordinator read the weekly page, and whether any drop time changed.

### Comparing against a manual tally

- **Sessions.** Each door gets at least 3: one in days 1–3, one in days 12–18, and one in the last week, with at least one in the evening peak. Keep going until the fridge has at least 100 tallied opens in total and the cabinet at least 50. If the cabinet can't reach 50, report its wider uncertainty rather than claiming 5 percent.
- **Recording.** Sync the tallier's phone to network time. Log each open with its minute and door, and nothing about the person. Tally sheets are raw event data: a key holder keeps them, they are never published, and they are deleted with raw rows at 90 days.
- **Matching.** A key holder matches tallied opens to device opens on raw events, within ±2 minutes after clock correction, not on hour buckets. That keeps clock error out of the count test.
- **Scoring.** Error = (misses + extras) / tallied opens. Use this, not the net difference, because misses and extras can cancel. Report timestamp error (median and maximum) separately.
- **Pass:** error of 5 percent or less on at least 100 fridge opens, no session over 10 percent, and no site visit caused by the device. A visit caused by an unannounced host Wi-Fi change is logged separately as a transport finding for v2, but it still counts against "no site visit."

### Stop

- **Stop at once and take both units down** on any privacy or data-access finding: plaintext anywhere, published raw rows, a live door signal, or install mode re-arming in the field.
- **Stop the field trial and start round 02** if any of these happen:
  - First-week tally error is over 15 percent after one re-seat.
  - Either door loses its contact or box twice.
  - By day 15, more than 25 percent of hours are gaps from device causes.
  - A known fault goes more than 24 hours without the email or the morning check reaching a person.
- **Anything in between is a fail:** keep the units running to the end for v2 data.
