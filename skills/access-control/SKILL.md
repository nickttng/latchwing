---
name: access-control
description: Make sure only named key holders can read raw events. Use when designing storage, upload, provisioning, or anything that moves or reads data.
---

# Access control

See decision 0006.

## Seal
1. At batch start, get a 256-bit data key from the hardware random source.
2. Wrap the data key to each key holder's public key and write that header to flash.
3. Encrypt each record with an authenticated cipher, using a counter as the nonce.
4. Keep the data key in RAM only. Wipe it when the batch closes. After a reboot, start a new batch.

Power loss doesn't lose sealed data, because the wrapped key is already in flash.

Use a well-known library. Never write your own crypto.

## Upload
1. Use TLS, but don't rely on it. Batches are already sealed.
2. Sign each batch with the device's identity key.
3. Clear a batch only on an acknowledgment signed by the server, naming the device and batch.
4. Check the signature with a public key stored on the device. The ack also carries the time.

## Provision
1. Generate the device identity key on the device, or on a key holder's machine.
2. Load the key holders' public keys and the ack public key.
3. Lock debug access and flash readout. Accept only signed firmware.
4. Give the device its own limited Wi-Fi login, on a guest network if there is one.

## Key holders
- Keep the list of names outside this repo. Only public keys are used in builds.
- Protect each private key with a passphrase, and keep an offline backup.
- To remove someone, re-provision with the new list and retire their key.
- If a device goes missing, revoke its identity key on the server.

## Check
- A flash dump shows no readable events or usable keys.
- A batch opens with each holder's key, and fails with any other key.
- A forged or replayed ack clears nothing.
- A device with an unknown identity key is rejected by the server.
- No open debug port, access point, or Bluetooth service remains.
- A scan of the repo finds no private keys or Wi-Fi logins.
