# 0005: Sealed enclosure and planned power

**Status:** Accepted

## Context
Chicago brings freeze/thaw, snow, humid summers, and condensation inside the fridge. Cold also cuts battery capacity.

## Decision
- Housing rated IP65 or better, with cable entries sealed and facing down, and a desiccant pack inside.
- Lithium primary cells, which hold up in the cold better than alkaline.
- Every build has a written duty-cycle budget with cold derating and a safety margin before it ships.

## Consequences
- Slightly higher part cost, much less maintenance.
- Battery life is a planned number, checked against field data.

## Rejected
- Mains power: needs an electrician and adds outdoor wiring risk.
- Unsealed project boxes: fail in the first wet season.
