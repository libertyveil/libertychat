# Mailbox layer

The mailbox layer is the foundation for asynchronous message delivery. It is heavily inspired by SimpleX's SMP design, refined and modernized.

## Concept

A **queue** is an anonymous, unidirectional message buffer hosted on a mailbox server:

- Identified by a random 32-byte ID
- Holds encrypted blobs in FIFO order
- Has a recipient (who reads + manages it) and a sender (who writes to it)
- Both roles authenticated by per-queue keypairs, not by user identity

## Per-conversation queues

A conversation between Alice and Bob requires **two queues** (one per direction):

- Queue A→B: Bob's incoming. Alice writes; **all of Bob's devices read.**
- Queue B→A: Alice's incoming. Bob writes; **all of Alice's devices read.**

Each queue can live on a different mailbox server, chosen by the recipient. Splits metadata across multiple servers — none can see the full picture of a user's social graph.

### Multi-device fanout

A user may have multiple devices (phone, laptop, tablet). All of those devices **subscribe to the same incoming queue**.

When the sender writes a message:
1. Encrypts the message **once** for the entire recipient (all devices) — see encryption details below
2. Uploads the encrypted blob **once** to the queue
3. Mailbox server stores the blob and notifies all subscribed devices
4. Each subscribing device pulls (or is pushed) the blob
5. Each device decrypts independently using its own key material

The sender does **not** encrypt N times for N recipient devices. One encryption, one upload, server-side fanout to subscribers.

This means:
- Sender bandwidth is O(message size), not O(message size × recipient devices)
- Storage on the mailbox server is O(message size), not O(× recipient devices)
- Receiver-side decryption is O(1) per device

### Multi-device key distribution

Each user's set of devices is treated as a **group** in MLS terms (RFC 9420):

- Alice has her own "device group" containing Phone, Laptop, Tablet
- Bob has his own "device group" containing his devices
- A 1:1 conversation between Alice and Bob is an MLS group whose members are **all of Alice's devices + all of Bob's devices**
- A group conversation is the same, just with more users contributing more devices

When the sender encrypts a message:
- The message is encrypted with the current MLS group's epoch key
- Every device in the group (including sender's other devices) can decrypt
- One ciphertext per message, regardless of total device count

Adding a device:
- User's new device generates its keypair and is signed by master identity
- Existing trusted device adds the new device to all relevant MLS groups via standard MLS Add proposals
- MLS handles epoch transition; the new device receives a Welcome with current group state

Removing a device:
- Existing device issues MLS Remove proposal + revocation certificate for the device
- New epoch derived; removed device's keys no longer valid
- All other devices continue uninterrupted

This is conceptually similar to how Matrix's Megolm rooms work, but standardized via MLS and applied to 1:1 conversations as well as groups.

## Operations

The protocol exposes a small set of server operations:

| Operation | Who | Purpose |
|---|---|---|
| `CREATE` | recipient | Allocate new queue with given keys |
| `WRITE` | sender | Append encrypted blob to queue |
| `READ` | recipient device | Fetch blobs from queue |
| `ACK` | recipient device | Confirm receipt by this device |
| `DELETE` | recipient | Remove queue entirely |
| `STATUS` | recipient | Get queue size, last activity |
| `SUBSCRIBE` | recipient device | Long-poll or push notifications for new arrivals |

Each operation is signed by the appropriate key. The server verifies the signature against the keys stored at queue creation time.

### Per-device subscription and ACK

Multiple devices of the same recipient can simultaneously subscribe and read from the queue. The server tracks acknowledgement **per subscribed device**:

- Each device sends its own `ACK` after successfully processing a message
- The server retains the blob until **all** subscribed devices have acknowledged
- Only then is the blob eligible for deletion

This guarantees that a message survives until every recipient device has had a chance to fetch it, while still bounded storage (expiry policy applies).

Device subscriptions are authenticated via the recipient sign key (which all devices possess via secure sharing) plus a per-device sub-key signed by the master identity. This way, the server can authorize multiple devices without exposing which is "primary".

## Queue creation flow

1. Recipient generates:
   - Random queue ID (32 bytes)
   - Recipient sign keypair (Ed25519): `rs_sec`, `rs_pub`
   - Sender sign keypair (Ed25519): `ss_sec`, `ss_pub`
   - Queue DH keypair (X25519+Kyber hybrid): `qdh_sec`, `qdh_pub`
2. Recipient sends to server: `CREATE queue=<ID> rs_pub sign_pub qdh_pub`
3. Server stores the tuple; returns OK.
4. Recipient bundles the connection details into an invitation:
   ```
   lvi://server-fingerprint@host:port/<queue_id>?ss=<ss_sec>&qdh=<qdh_pub>
   ```
5. Recipient shares the invitation URL out-of-band with the sender.
6. Sender's client parses the URL, generates its own DH keypair, and writes the first message containing its DH public key + initial encrypted payload.

Once both parties have exchanged initial key material via the queue, they bootstrap an **MLS group** (with all of both parties' devices as members). All subsequent message contents are encrypted with the MLS group's epoch keys, not with raw Diffie-Hellman shared secrets.

The mailbox-layer crypto (queue handshake) protects against the server learning queue membership; the MLS-layer crypto protects message content end-to-end across all participating devices.

## Encryption layers

A message in transit has three encryption layers:

```
┌──────────────────────────────────────┐
│ TLS 1.3 / QUIC (transport)           │  ← protects against network eavesdroppers
│ ┌──────────────────────────────────┐ │
│ │ Mailbox-layer crypto (NaCl box)  │ │  ← wraps blob for queue (sender ↔ queue owner)
│ │ ┌──────────────────────────────┐ │ │
│ │ │ Application-layer (MLS)      │ │ │  ← end-to-end encrypted, all recipient devices
│ │ └──────────────────────────────┘ │ │
│ └──────────────────────────────────┘ │
└──────────────────────────────────────┘
```

- **Transport** encrypts the wire between client and mailbox server.
- **Mailbox-layer crypto** wraps the blob for the queue. Independent of user identity; rotates per-queue.
- **Application-layer** uses MLS for both 1:1 (treated as 2-user, multi-device group) and groups. All recipient devices share the MLS epoch state and decrypt the same ciphertext.

This unifies 1:1 and group encryption under a single standardized protocol (MLS RFC 9420). The legacy distinction between Double Ratchet (1:1) and group ratchets (Megolm-style) is collapsed into one model.

## Server scalability

A mailbox server is a thin process:

- In-memory queue index (queue ID → metadata)
- Disk-backed message store (sled or RocksDB for embedded simplicity)
- TLS 1.3 + QUIC listener
- TCP fallback listener

Resource estimates (back-of-envelope):

- 10,000 active users with ~5 conversations each = 50,000 active queues
- Average 10 messages/day in flight per queue, 1 KiB each = 500 MB/day of message storage
- Steady-state RAM: ~100 MB
- CPU: dominated by TLS handshakes; modest

A single small VPS (1 GB RAM, 1 vCPU) can serve a small community of hundreds of users.

## Message expiry

Server-side message retention:
- Default: 30 days unread, then deleted
- Configurable per-server (operator policy)
- Successfully delivered messages: deleted on ACK from recipient

This bounds storage growth and provides server-side privacy: even with subpoena, expired messages are unrecoverable.

## Queue migration

A user can migrate a queue from one mailbox server to another:

1. New server: create new queue with same keys
2. Existing recipient client: notify all peers of the migration
3. Old server: stop accepting writes, forward existing messages
4. Peers: switch their write target to new queue

This solves SimpleX's queue-binding-to-server problem (you can change provider without losing contacts).

## Multi-server distribution

A user can distribute their queues across multiple mailbox servers:

- Friends queue → server A (cheap VPS)
- Work queue → server B (paid managed)
- Anonymous contact queue → server C (Tor-only)

No server sees more than a fragment. Client coordinates which queues live where.

## Differences from SimpleX SMP

| Aspect | SimpleX SMP | LibertyChat Mailbox |
|---|---|---|
| Wire protocol | Custom binary over TCP | QUIC + Protobuf wire format |
| Queue auth | Ed25519 signatures | Ed25519 signatures (same) |
| Hybrid PQ in queue handshake | Recently added | First-class from start |
| Migration support | Limited | First-class |
| Operator economy | None / community | None / community (same) |
| Server implementation | Haskell | Rust |
