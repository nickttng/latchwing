# Addendum 4, round 01

This answers `review-4.md` and uses the reviewer's own fix text. Order of authority: addendum-4, then addendum-3, then addendum-2, then addendum, then verdict, then revision. Date: 2026-11-03.

**Status:** `review-5.md` asked for two one-sentence fixes, to X1 and K2. Both are applied below as the reviewer worded them, and the reviewer said that with them the round passes with conditions.

Hard constraints: unchanged. Cellular is the user's preference, not a brief constraint (M10).

## Blocking

- **K1. `no_key`, replacing R1's example in addendum-3:**
  - `low_batt_skip` skips association and transfer only. The key is still drawn at RF start in every window.
  - `no_key` applies only when RF start fails or resets the chip. While `no_key` holds, no slots of any kind are written.
  - The first health row under the next key records the `no_key` range in powered hours.
  - When writing resumes, the pin is read once and at most one change is logged, as in S3.
  - **Gates:**
    - Added to the sag gate: a window that fires `low_batt_skip` still records door events under a fresh key.
    - A forced RF-start failure gives one `no_key` row with the right range and no slots inside it.
- **K2. A full key log, replacing R3's "keep the current key":**
  - A full key log never extends a key past its window or the backstop.
  - Instead, the oldest key-log sector is erased, which crypto-erases its unsent slots. That matches 0003's oldest-first overwrite.
  - The report shows the result as a gap with the cause `keylog_full`.
  - **In addendum-3's R3,** "except under the 90-day rule" now reads "except under the 90-day rule or when the key log is full (K2, `keylog_full`)".
  - **Gate,** added to R3's: with the key log full, no key covers more than one window.
- **K3. Expiry margin, replacing addendum-3's T3:**
  - A key-log sector is erased when its *oldest* entry is older than 90 d × (1 − r) − 1 d, in powered hours.
  - Every reset that isn't a deep-sleep wake adds 1 h to the powered-hour count, so lost heartbeats push the count up and erasure errs early.
  - `r` is the worst raw slow rate across every board's own drift-gate run (M8).
  - **Gates:**
    - A sector whose oldest entry is past the limit is erased, even if its newest entry isn't.
    - 10 power-on resets raise the count by at least 10 h.

## Storage-spec requirements, added to the deferred R2, R3 and R5

- **Q1, R2 gate:** wrap the ring twice on the bench. Every `overwritten` range matches the slots lost. The layout code asserts that no slot or key-log entry crosses a sector.
- **Q2, R3:**
  - The reset-reason row is deferred to the next hourly row.
  - Each key-log entry is stamped with the heartbeat count at which its batch began, not the count when the entry was written.
  - **Gate:** 200 resets with no door activity add no key-log entries.
- **Q3, R5:**
  - A backlog is sent in chunks of at most the largest size step. Normal uploads keep the size steps, so T5's budget stands.
  - The largest step is set so that one upload completes within the 20 s cap at the weakest RSSI that passes the W2 gate.
  - Wrapped keys count inside the padded size.

## Decision text

- **X1. 0006 90-day line, in full:** "Raw data is deleted on these schedules. On the device, that is 90 powered days less the margin (K3). Pulled copies are deleted 90 days from their oldest record. On the server, copies are deleted 90 days after upload, so a copy can last up to 90 days plus the length of an outage. This last case departs from the original line, and the departure is accepted."
- **X2. 0004 gap list:** adds `no_key` and `keylog_full`, next to `expired`.
- **X3. T1, reworded:** "...can't run again on this board." Also: "USB prints the identity public key once, at enrollment."

## Non-blocking items taken

- **M1, R6 hardening:**
  - Check `usb_serial_jtag_is_connected()` after a settle delay of about 1 s.
  - **Gate:** an armed unit with cells fitted and USB connected at power-on starts no window.
  - The battery divider sits on the cell side of the 1N5817.
- **M2, public page:** keep suppressing the next-smallest cell until the suppressed cells include one of 3 or more.
- **M3, hotspot:**
  - The hotspot uses a dedicated rescue SSID, turned on only for a planned rescue. A holder's everyday hotspot is never configured.
  - `rssi` is read only at upload attempts.
- **M4, W2 logger:**
  - After the readout and before provisioning, the board gets a full chip erase (`esptool.py erase_flash`).
  - The test route is a separate Worker with no access to sealed-batch storage, and it is deleted after the gate.
  - The 0003 text says "attempt times and results".
- **M5, rotation window:** R1's window is one scheduled window: 3 h, or 6 h if the drift test moves uploads. At 6 h, about 7,206 slots is still under the half-ring backstop.
- **M6, T2 wording:** "only as totals between resets".
- **M7, naming:** in T8, "R2 storage" means Cloudflare R2, not review item R2. "T3" there means addendum T3.
- **M9, panic reset:** folded into R1's gates. A panic reset produces a new key, and the old key is wiped at boot.
