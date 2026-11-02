---
name: event-driven-wake
description: Sleep until a door changes state, wake on a pin interrupt, and debounce to one clean event. Use when choosing a sensor, wiring it, or writing wake handling.
---

# Event-driven wake

See decision 0001.

## Technique
1. Mount a reed switch or hall sensor on the frame, with a magnet on the door.
2. Wire the sensor to a pin that can wake the chip from deep sleep.
3. Enter deep sleep with wake-on-change for that pin.
4. On wake: debounce, read the state, append an event, and go back to sleep.

## Sensor choice
| Sensor | Use when | Watch out for |
|---|---|---|
| Reed switch | Default; near-zero idle power | Needs tight magnet alignment; can stick |
| Hall sensor | Gap may vary, or the door flexes | Small idle current; check it in the budget |

## Debounce
1. On wake, record the raw pin level.
2. Wait a settle window (start at 50 ms).
3. Re-read. If the level is unchanged and differs from the last recorded state, emit one event.
4. Ignore any change within a lockout window after an event (start at 500 ms) to absorb door rattle.

Only emit a state *change*. Two `open` events in a row is a bug.

To tune, count events against a manual tally of 20 opens and closes. Extra events: lengthen the lockout. Missed events: shorten the settle window.

## Check
- The sensor-to-magnet gap stays within spec with the door shut.
- A steel door doesn't weaken the magnet's field too much.
- Idle current measured in deep sleep matches the budget.
- Mounted so the door can't snag it or knock it loose.
- Fridge doors: sensor is rated for cold and condensation.
