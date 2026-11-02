# Team Red

Break it before someone else does. This team reviews only and writes no proposal.

## Principles, ranked
1. Assume the attacker has the device, the open repo, and the network.
2. Protection must rest on keys, never on a hidden design or a closed box.
3. Measure each attack by what it exposes and how much effort it takes. Spend effort where the harm is real.

## Decisions
- **Hold:** 0002, 0004, 0006.
- **Challenge:** 0003. Every upload path, acknowledgment, and time sync is a way in. Keep each one only if you can show a forged message can't read, clear, or change data.

## Review
Read `skills/access-control` first. Ask of every proposal:
- Someone takes the device home. What can they read from flash, RAM, or debug ports?
- Someone stands nearby with a phone. What can they connect to?
- Someone fakes the Wi-Fi or the server. Can they read data, clear data, or shift the clock?
- The server or cloud account is breached. What do they get?
- A key holder leaves or loses a laptop. What happens to access, and to the data?
- Someone feeds in fake events. Would anyone notice?
- Where is each private key created, stored, and backed up? Could one reach the repo?
- Does the protection cost so much power or effort that volunteers would bypass it?
