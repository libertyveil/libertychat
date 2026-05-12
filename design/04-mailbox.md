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

- Queue A→B: Bob's incoming. Alice writes; Bob reads.
- Queue B→A: Alice's incoming. Bob writes; Alice reads.

Each queue can live on a different mailbox server, chosen by the recipient. Splits metadata across multiple servers — none can see the full picture of a user's social graph.

## Operations

The protocol exposes a small set of server operations:

| Operation | Who | Purpose |
|---|---|---|
| `CREATE` | recipient | Allocate new queue with given keys |
| `WRITE` | sender | Append encrypted blob to queue |
| `READ` | recipient | Fetch blobs from queue |
| `ACK` | recipient | Confirm receipt; server may delete |
| `DELETE` | recipient | Remove queue entirely |
| `STATUS` | recipient | Get queue size, last activity |
| `SUBSCRIBE` | recipient | Long-poll or push notifications for new arrivals |

Each operation is signed by the appropriate key. The server verifies the signature against the keys stored at queue creation time.

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

Once both parties have exchanged DH public keys via the queue, they derive a shared secret (Diffie-Hellman) and bootstrap a Double Ratchet session. From then on, all message contents are encrypted with the ratchet, not just queue-layer crypto.

## Encryption layers

A message in transit has three encryption layers:

```
┌──────────────────────────────────────┐
│ TLS 1.3 / QUIC (transport)           │  ← protects against network eavesdroppers
│ ┌──────────────────────────────────┐ │
│ │ Mailbox-layer crypto (NaCl box)  │ │  ← wraps blob for the queue (sender ↔ recipient)
│ │ ┌──────────────────────────────┐ │ │
│ │ │ Application-layer (Double R.) │ │ │  ← end-to-end encrypted message content
│ │ └──────────────────────────────┘ │ │
│ └──────────────────────────────────┘ │
└──────────────────────────────────────┘
```

The mailbox-layer crypto is independent of identity and rotates per-queue. The application-layer crypto rotates per-message via the Double Ratchet.

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
