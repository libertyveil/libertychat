# Identity

## Goals

- Users own their identity cryptographically. No registration with any provider.
- Optional human-readable discovery (`@user@domain`) without surrendering control.
- Multi-device support: one logical identity, multiple physical devices.
- Identity is recoverable if a device is lost (with user-managed backup material).

## Identity primitive

A **user identity** is a long-term keypair pair:

- **Signing keypair**: Ed25519 + CRYSTALS-Dilithium (hybrid, post-quantum resistant)
- **Key-exchange keypair**: X25519 + CRYSTALS-Kyber (hybrid, post-quantum resistant)

These are generated locally on first install. The combined public material forms the **Identity Public Key (IPK)**. The IPK is the user's globally-unique cryptographic identifier.

Format: base32-encoded concatenation of the four public components, plus a checksum and version byte. Roughly 200 characters as a string. Display-friendly via QR code or chunked grouping.

## Devices and sub-keys

Each physical device (phone, laptop, tablet) generates its own **device keypair** and is signed by the user's master identity. The result is a **device certificate** containing:

- Device public key
- Device label (human-readable, e.g. "Alice's iPhone")
- Issued-at, expires-at timestamps
- Signature by master identity

When a contact receives a message from a device, they verify the device certificate against the known master identity. If valid, the device is trusted as belonging to that user.

### Device group model

A user's set of devices is modeled as an MLS group internally:

- The "user's device group" contains all of the user's active devices as members
- The master identity is the group's anchor (signs all member additions/removals)
- Each conversation (1:1 or group) is itself an MLS group whose members are all the participating users' devices

This means a 1:1 conversation between Alice (3 devices) and Bob (2 devices) is technically an MLS group with **5 members**. When Bob sends a message:
- Encrypted once with the conversation's MLS epoch key
- Delivered to Alice's incoming mailbox queue (a single upload)
- All of Alice's subscribed devices receive the same ciphertext
- Each device decrypts independently using its own MLS state

### Multi-device message reception

All of a user's devices **subscribe to the same incoming mailbox queue** for a given conversation. The mailbox server fanout means one upload reaches all devices.

This is fundamentally different from SimpleX's master-slave model:
- **SimpleX**: one "master" device holds the queue subscription; "linked" devices are remote views that only work when the master is online
- **LibertyChat**: every device is an equal first-class subscriber; any device works independently, even if others are offline

### Adding a new device

1. New device generates device keypair.
2. User authenticates from existing trusted device (QR scan or short-code).
3. Existing device:
   - Signs a device certificate for the new device
   - Adds the new device as a member to every active conversation's MLS group via standard MLS `Add` proposals
   - The MLS commit triggers an epoch transition
4. New device receives Welcome messages with current group state for each conversation
5. From this point on, the new device participates equally in all conversations

### Removing a device

1. User issues an MLS `Remove` proposal for the device, on every conversation it's a member of
2. Commits trigger epoch transitions; new epoch keys exclude the removed device
3. Revoked device's certificate is published to contacts so it's no longer trusted for new messages
4. Old messages decrypted with prior epoch keys remain accessible on the removed device until the local DB is wiped (server can't enforce wipe)

This solves SimpleX's multi-device pain: any device can act independently, all are bound to one identity, key compromise affects only one device.

## Recovery

Master identity is critical — losing it means losing the ability to add new devices and prove identity to contacts. Recovery options:

1. **Mnemonic phrase**: BIP-39-style 24-word phrase derived from the master sec-key. User writes it down on paper or stores it in a password manager.
2. **Sharded backup (Shamir's Secret Sharing)**: Master key split into N shares, T of which can reconstruct it. User distributes shares to trusted contacts or storage locations.
3. **Hardware token**: Master keys stored on a YubiKey or similar; lost-device scenario means using the hardware token to authorize a new device.

Recovery does NOT restore message history. History lives on the user's mailbox server (encrypted) and on individual devices. A new device starts fresh and rebuilds history from the mailbox.

## Discovery

Identity discovery is **optional and decentralized via DNS**.

A user with the domain `example.com` can publish an **identity descriptor** as a DNS TXT record on `_libertyveil.example.com` and as a JSON document at `https://example.com/.well-known/libertyveil/identity.json`.

Descriptor contents:

```json
{
  "version": 1,
  "user": "alice",
  "ipk": "base32-encoded-master-public-key",
  "mailbox_servers": [
    "lvm://server1.example.com:5223",
    "lvm://backup.alice.com:5223"
  ],
  "discovery_servers_optional": [
    "lvd://discovery.example.com:5224"
  ],
  "valid_until": "2027-12-31T23:59:59Z",
  "signature": "ed25519+dilithium-signature-of-above-fields"
}
```

When user A wants to contact `@alice@example.com`:
1. A's client queries `_libertyveil.example.com` TXT record (or fetches well-known JSON).
2. Validates the signature against the published `ipk`.
3. Connects to one of the listed mailbox servers to deliver an introduction message.

The discovery layer never sees content or correlates conversations — it only publishes the user's existence and contact endpoints.

This pattern echoes Mail's MX records and HTTPS's `.well-known` mechanism. Users with their own domain control their identity entirely.

## Anonymous identities

Users without a domain can simply not publish a discovery record. They share their IPK out-of-band (QR code, paste into Signal, etc.). This matches the SimpleX model and is the recommended default for users who want maximum anonymity.

## Summary of choices

| Aspect | Choice | Why |
|---|---|---|
| Master keys | Ed25519+Dilithium and X25519+Kyber, hybrid | Post-quantum readiness from day one |
| Device model | Master signs per-device certificates | Multi-device without master-key sharing |
| Identifier | Cryptographic public key, no name | Cannot be revoked or banned by a registry |
| Discovery | Optional DNS-based, opt-in | Leverages existing infrastructure users already own |
| Recovery | Mnemonic, Shamir, or hardware token (user choice) | No service-level account recovery |
| Multi-identity | Multiple independent identities supported per user | Persona separation when desired |
