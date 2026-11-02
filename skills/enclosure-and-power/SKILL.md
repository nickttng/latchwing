---
name: enclosure-and-power
description: Seal the device against water, cold, and condensation, and plan battery life up front. Use when choosing a board, battery, or upload schedule, before any outdoor install, and after any failure.
---

# Enclosure and power

See decision 0005.

## Checklist
- [ ] Housing rated IP65 or better
- [ ] Cable glands sealed and pointing down
- [ ] Desiccant pack inside, replaced at each battery change
- [ ] Lithium primary cells
- [ ] Sensor and magnet sealed or potted
- [ ] Mounted clear of water spray and snow pile-up
- [ ] Door seals and vents unobstructed

## Tests
- Freezer for 24 hours, then compare events against a manual tally.
- One week outdoors alongside a manual tally.

## Power budget
1. List the states: deep sleep, wake and log, upload. Include encryption and signing time, measured on the real chip.
2. For each, note current (mA) and time per day (h).
3. Daily mAh = sum of mA x h.
4. Life in days = capacity x cold derating (0.5 for freezing) / daily mAh / 1.3 margin.

Levers, in order of impact:
1. Deep-sleep current
2. Upload frequency and radio time
3. Wake time per event

Output a table of states with the final estimate and its assumptions. Compare it against field data each month.
