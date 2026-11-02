# 0006: Only named key holders can read raw events

**Status:** Accepted

## Context
The device sits in public 24/7 and this repo is open, so anyone can reach the hardware and read how it works. Door times are low risk, but they still show when someone was at the fridge. Protection can't depend on a locked box, a hidden design, or a trusted network.

## Decision
- Events are sealed on the device to the public keys of a short list of named people. Only their private keys can open them.
- The device holds no key that can read sealed batches. Neither does the server, the network, or a USB pull.
- Each batch uses a new random data key, kept only in RAM and wiped when the batch closes.
- The device accepts only signed acknowledgments, so a fake one can't clear data.
- Each device has its own identity key, created at provisioning. No secrets go in this repo.
- At least two key holders, so one lost key doesn't lose the data, and no more than three.
- Raw events are deleted after 90 days. Published reports follow 0004 and are not restricted.

## Consequences
- A stolen device, server, or USB copy reveals no sealed batches.
- The open batch is the exposed window. It is limited by debug lock and upload frequency.
- If every private key is lost, the data is gone. That is the intended trade-off.
- Removing a key holder takes effect at the next provisioning. They keep what they already read.
- Provisioning adds a build step, and the chip needs a hardware random source and a debug lock.
- Upload timing and size can still hint at how busy a door is.
- This protects stored data, not the device. Someone can still break or take it, and swapped firmware could see new door events. A missing device shows up as a gap in the data.

## Rejected
- Encrypting only in transit: leaves plain data on the device, the server, and USB.
- One shared key stored on the device: one stolen device exposes everything.
- Passwords or open wireless access on the device: public hardware invites guessing.
- A locked box or hidden design as protection: the box is public and the design is open.
- A separate secure-element chip for v1: adds cost and parts for little gain over public-key sealing.
