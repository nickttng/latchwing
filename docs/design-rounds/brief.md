# Design brief

Every team works from this page. Edit it between rounds as real facts come in.

## Goal
Choose the best first device to test in real life. It is not the final product. The best v1 is the one most likely to produce trustworthy door counts from a real fridge, and to teach us the most for v2.

## The trial
These are starting defaults. Change them to fit the pantry.
- One 24/7 community fridge door and one cabinet door in Chicago.
- Installed by a volunteer using only the written steps.
- 30 days unattended, starting this winter.
- Parts budget of $50 per device.
- Success: counts within 5 percent of a manual tally, and no site visit needed to keep it running.

## Hard constraints
No team may challenge these:
- No cameras, audio, presence sensing, or anything that identifies a person.
- No published raw events and no live "door is open" feeds.
- Only named key holders can read raw events. A stolen device, a breached server, or a tapped network must not expose sealed data.

Everything else can be challenged, including accepted decisions in `docs/decisions/` and how 0006 achieves that last outcome. A team that departs from a decision must name it and say what replaces it.

## Site facts
Not yet known. Teams must state what they assume and what changes if they are wrong.
- Wi-Fi reach at the fridge
- Whether there is a spare outlet, and who controls it
- Door material, and where a device can mount
- Distance between the fridge and the cabinet
- Who the two or three key holders are

## Open questions
Every proposal must answer these:
1. With event-driven wake, how does the data tell a dead device from a quiet door? Decision 0004 requires gaps to be shown as gaps.
2. Decision 0005 rejected mains power, but a fridge is already plugged in. Does that still hold?
3. Is it one device for both doors, or one per door?
4. It sits in public 24/7. What happens when someone pulls at it, and how do we keep it from looking like surveillance?

## Scorecard
Privacy and data access are pass or fail. A design that fails either is out. Score the rest from 1 to 5, then multiply by the weight.

| Criterion | Weight |
|---|---|
| Count accuracy against a manual tally | 3 |
| Survives the trial unattended | 3 |
| A volunteer can build and install it | 2 |
| Answers a real pantry question | 2 |
| Cost and part availability | 1 |
| Someone else can rebuild it from the repo | 1 |

## Learning from the field
After a trial, add what happened here: what broke, what the tally showed, and what volunteers said. The next round starts from it.
