# Cryptography

## Principles

1. **Use audited primitives.** No custom cryptography. Build only protocol-level composition on top of well-reviewed building blocks.
2. **Hybrid post-quantum from day one.** Every classical primitive is paired with a post-quantum equivalent.
3. **Forward Secrecy and Post-Compromise Security** for every conversation.
4. **Standardized group encryption** via MLS (RFC 9420).

## Building blocks

### Symmetric encryption
- **ChaCha20-Poly1305** (AEAD)
- AES-256-GCM as alternative for hardware-accelerated platforms

### Hashing
- **BLAKE3** for general purposes (fast, parallel)
- **SHA-256** for compatibility where mandated by external standards

### Key derivation
- **HKDF-SHA-256** for KDF chains
- **Argon2id** for password-based KDF (mnemonic recovery)

### Signatures (hybrid)
- **Ed25519** + **CRYSTALS-Dilithium** parallel
- A signed message includes both signatures
- Verification requires both to validate
- A break in one does not compromise the other

### Key exchange (hybrid)
- **X25519** + **CRYSTALS-Kyber** parallel
- Shared secret = HKDF(X25519-output || Kyber-output)
- Used in handshake and ratchet steps

### Random number generation
- OS-provided (`/dev/urandom`, `getrandom()`, `BCryptGenRandom()`)
- Audited Rust wrapper (`rand_core::OsRng`)

## Protocol layers

### One-to-one messaging: Double Ratchet

Same as Signal Protocol, with hybrid PQ key exchange (PQXDH-style) replacing classical X3DH.

- Initial handshake derives a shared root key from both parties' identity and ephemeral keys
- Per-message keys derived via Double Ratchet (symmetric ratchet + DH ratchet)
- Forward Secrecy: each message encrypted with a unique key, deleted after use
- Post-Compromise Security: DH ratchet recovers from key compromise after the next exchange

### Group messaging: MLS (RFC 9420)

Standardized group key agreement protocol. Properties:

- **Logarithmic message overhead** for adding/removing members in groups of any size
- **Continuous key rotation** as members join and leave
- **Forward Secrecy** for past messages
- **Post-Compromise Security** after the next epoch
- **Authentication** of all group operations

We use the Rust `mls-rs` library (AWS open source implementation, formally analyzed).

### Mailbox queue auth

Operations against the mailbox server (read, write, manage queue) are signed by per-queue signing keys, separate from user identity. This prevents the server from learning the user's identity even when serving their queues.

- **Recipient sign keypair**: stays with queue owner; signs read/manage requests
- **Sender sign keypair**: shared with the contact in the invite link; signs write requests
- Both are Ed25519 (PQ not strictly necessary at this layer; queue auth is short-lived per-queue)

### File chunks

Files are split into chunks of fixed size (e.g. 64 KiB), each encrypted independently with ChaCha20-Poly1305 using a per-chunk key derived from a master file key.

The master file key is shared via the chat channel. The file server stores the encrypted chunks blindly.

## Threat model coverage

| Adversary | Capability | LibertyChat protection |
|---|---|---|
| Mailbox server operator | Reads all blobs at the server | Cannot decrypt content (E2EE) |
| Network observer (ISP) | Sees TCP connections, sizes, timing | TLS 1.3 transport encryption; onion routing optional |
| Malicious contact | Has shared secret with you | Can read messages with you (by definition); cannot impersonate you to others (signatures) |
| Quantum computer (future) | Can break ECDH and EdDSA | Hybrid PQ keys remain secure |
| Compromised device | Reads local keys | Forward Secrecy: only future messages affected; older keys deleted |

## What we deliberately do not protect against

- **Endpoint compromise**: if the user's device is compromised (malware, physical access), all guarantees break. Standard limitation of any messenger.
- **Coercion**: if the user is forced to reveal their secrets, no cryptography helps.
- **Side-channels in the OS**: keystroke logging, screen capture, etc. are out of scope.

## Cipher agility

The protocol carries algorithm identifiers in every message. Future versions can introduce new primitives (e.g. Kyber successor, Dilithium successor) without breaking compatibility — clients negotiate the strongest mutually-supported set.

## Audit plan

Pre-1.0 release requires:
- External audit of `libertychat-core` cryptographic implementation (Trail of Bits, Cure53, or equivalent)
- Formal model of the protocol (Tamarin or ProVerif)
- Open security bounty program

No production deployment without audit completion.
