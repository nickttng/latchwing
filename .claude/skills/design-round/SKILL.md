---
name: design-round
description: Run a full design round. Three teams design separately, critique each other, and revise, then a judge picks one v1 device to trial.
argument-hint: "[focus for this round]"
disable-model-invocation: true
---

# Design round

Focus for this round, if any: $ARGUMENTS

Design teams are `scrappy`, `hardened`, and `pantry`. `red-team` reviews but doesn't design. Their charters are in `docs/teams/`.

Run this from the main session, because subagents can't start other subagents. Create the next numbered folder, such as `docs/design-rounds/01/`. If an earlier round exists, tell every agent to read its `verdict.md`, along with the brief. Pass the focus to every agent.

## Steps
Run the agents in each step in parallel. Wait for all of them before starting the next step. If an expected file is missing, run that agent again before moving on.

1. **Propose.** Run one `team-designer` per design team with the task "propose". Each works blind: tell it not to open any other team's file.
2. **Critique.** Run one `team-reviewer` per team, including `red-team`. Design teams review the other two proposals. `red-team` reviews all three.
3. **Revise.** Run one `team-designer` per design team with the task "revise". Each reads every critique of its proposal, including the red team's.
4. **Judge.** Run one `judge`.
5. **Review.** Run `design-review` on `verdict.md`, and have it write `review.md` in the round folder. Departures listed in the verdict are expected, but each one needs a decision change there. Any change to the hard constraints in the brief is a failure.

## Finish
Report the pick, the strongest dissent, the review result, and the site facts to check first.

Don't edit `docs/decisions/`, the charters, or the brief. Those changes are the user's call.
