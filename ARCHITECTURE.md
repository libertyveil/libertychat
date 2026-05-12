# Architecture Overview

This document describes the high-level system architecture. Detailed specifications for each subsystem live in [`design/`](design/).

## Layered model

```
┌─────────────────────────────────────────────────────┐
│  Application clients (CLI, Desktop, Mobile)         │
├─────────────────────────────────────────────────────┤
│  Client SDK (Rust core, FFI bindings to Swift/Kotlin) │
├─────────────────────────────────────────────────────┤
│  Protocol layer                                     │
│    - Identity (DID-style keypairs, device certs)    │
│    - Conversations (MLS for all, 1:1 and groups)    │
│    - Realtime (WebRTC + DTLS-SRTP)                  │
│    - Mail (long-form persistent messages)           │
├─────────────────────────────────────────────────────┤
│  Transport layer                                    │
│    - Mailbox (asynchronous queue server)            │
│    - File store (chunked encrypted blobs)           │
│    - Realtime media (peer-to-peer or relay)         │
│    - Optional onion routing (Tor / Nym integration) │
├─────────────────────────────────────────────────────┤
│  Network primitives                                 │
│    - QUIC + Noise / TLS 1.3                         │
│    - libp2p-style NAT traversal                     │
└─────────────────────────────────────────────────────┘
```

## Components

### Identity layer

Each user generates a long-term **identity keypair** (Ed25519 + Dilithium hybrid) on first install. The public key is the user's cryptographic identity. There is no global username registry; identity is shared out-of-band via invitation links or QR codes.

Optionally, users can publish a self-signed **identity descriptor** to a discovery server (DNS, well-known URL on a domain they control) so others can find them by `@user@domain`. This is opt-in and does not affect the protocol.

See [`design/01-identity.md`](design/01-identity.md).

### Mailbox server

Asynchronous messages are stored on **mailbox servers**: dumb relays that hold encrypted blobs in queues. Each conversation uses dedicated, randomly-named queues. The server cannot read content (encrypted) and cannot link queues (no global identifier).

**One queue per conversation, multiple recipient devices subscribe.** The sender uploads each message once; the server fans the encrypted blob out to all subscribing devices of the recipient. This makes multi-device a first-class feature without multiplying sender bandwidth or server storage.

Mailbox servers are trivially self-hostable: a single Rust binary, no database server (sled or RocksDB embedded). One server can serve thousands of users.

See [`design/04-mailbox.md`](design/04-mailbox.md).

### Realtime layer

Audio and video calls use WebRTC with DTLS-SRTP encryption. Connection setup goes through the mailbox layer (signaling messages); media flows directly peer-to-peer when possible, falling back to a TURN relay for restrictive NAT.

LibertyChat ships its own TURN server implementation. Users can run their own or use community ones. TURN sees only encrypted media (E2EE end-to-end).

See [`design/05-realtime.md`](design/05-realtime.md).

### Mail layer

Long-form persistent messages (think email-style) live in a separate channel from realtime chat. Stored encrypted on the user's mailbox server. Threading, attachments, search are first-class.

This is what differentiates LibertyChat from pure messengers: a unified inbox for both ephemeral chat and persistent correspondence.

See [`design/06-mail.md`](design/06-mail.md).

### Group chats (MLS)

Group conversations use MLS (Messaging Layer Security, RFC 9420) for efficient key agreement at scale. Groups of hundreds to thousands of members are supported with logarithmic message overhead.

See [`design/07-groups.md`](design/07-groups.md).

### Privacy-routing layer

Users can choose per-conversation routing:

- **Direct** — client connects straight to mailbox server (lowest latency, IP visible to server)
- **Onion** — traffic routed through Tor or Nym mix-net (higher latency, IP hidden)
- **Federated** — message hops through user's home server, then to recipient's home server

See [`design/08-privacy-model.md`](design/08-privacy-model.md).

## Trust boundaries

| Component | Trusted to | Not trusted to |
|---|---|---|
| Mailbox server | hold encrypted blobs and deliver them | read content, link conversations |
| TURN/relay server | forward encrypted media bytes | decrypt media, observe content |
| Discovery server | resolve `@user@domain` to public key | impersonate users (signed records) |
| Other peers | participate in MLS group operations | send unauthorized messages |
| User's local device | hold private keys | n/a (root of trust) |

If any non-local component is compromised, content remains encrypted. The compromise reveals only metadata (who connected when, message sizes, timing).

## Implementation language

Rust for the core protocol library (`libertychat-core`) and all server components. Native UI bindings:

- iOS: Swift, calling into `libertychat-core` via FFI
- Android: Kotlin, calling into `libertychat-core` via JNI
- Desktop: Rust + Tauri or Iced (TBD)
- CLI: Rust binary

Rust chosen for memory safety, performance, deterministic resource use, and strong cryptographic library ecosystem (RustCrypto, mls-rs, libp2p, age).

See [`design/09-tech-stack.md`](design/09-tech-stack.md).

## Repository layout (planned)

```
libertyveil/
├── libertychat              ← protocol spec + concept docs (this repo)
├── libertychat-core         ← Rust protocol library
├── libertychat-server       ← mailbox server daemon
├── libertychat-relay        ← TURN/onion relay (optional)
├── libertychat-cli          ← terminal client
├── libertychat-desktop      ← GUI client
├── libertychat-mobile-ios   ← iOS app
├── libertychat-mobile-android ← Android app
└── libertychat-docs         ← user-facing documentation
```
