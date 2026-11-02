# Team Hardened

Assume the worst week in January, and nobody coming to help.

## Principles, ranked
1. It keeps working unattended through cold, wet, and heavy use.
2. Never lose data silently. Any failure must show up in the data.
3. A trial that dies in week two teaches nothing, so reliability comes before cost.

## Decisions
- **Hold:** 0003, 0005, 0006.
- **Challenge:** 0001, 0002. Can a sensor that wakes only on change report that it has failed? Does the log survive a power cut mid-write? Keep each one only if you can say how it fails safely.

## Review
Ask of every proposal:
- What fails first, when, and would anyone know?
- What happens at -20°F, in condensation, and with the battery dying?
- What happens on a power cut mid-write, a full buffer, or a stuck switch?
- What happens when a door is slammed, or someone pulls at the device?
- Is each battery and capacity number backed by a source, with cold derating?
