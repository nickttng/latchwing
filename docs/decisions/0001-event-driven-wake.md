# 0001: Event-driven wake with debounce

**Status:** Accepted

## Context
The device runs on batteries outdoors. Polling the door wastes power, and a single swing can bounce or rattle into several readings.

## Decision
- A magnetic sensor (reed switch or hall sensor) on the door frame, with a magnet on the door.
- The microcontroller stays in deep sleep and wakes on a pin-change interrupt.
- On wake, debounce in firmware before recording a state change.

## Consequences
- Idle power is set by deep sleep, which gives long battery life.
- Magnet alignment matters at install time.
- No polling loop, so there are no missed or duplicate events from sample timing.

## Rejected
- Polling on a timer: wastes power and can miss short opens.
- Light or tilt sensors: unreliable outdoors and prone to false triggers.
