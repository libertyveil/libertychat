# Vision

## Problem statement

Today's messaging landscape forces a trade-off:

- **Mainstream apps (WhatsApp, iMessage, Telegram)** offer good usability but require trust in a single corporate operator. Metadata is collected, the service can be coerced or shut down, and users cannot leave with their data.
- **Privacy-focused apps (Signal, Element)** improve on encryption but still depend on central servers (Signal) or are too heavy and complex to self-host comfortably (Matrix/Synapse).
- **Self-hostable apps (XMPP/Snikket, Matrix, SimpleX)** give control but each comes with friction: weak mobile A/V (XMPP), expensive infrastructure (Synapse), poor multi-device sync (SimpleX), feature gaps everywhere.
- **Pure P2P apps (Jami, Tox, Briar)** eliminate the server but lose asynchronous delivery, mobile push reliability, or both.

No existing system provides simultaneously:

1. Full messaging features (chat, audio, video, files, persistent mail, group chats)
2. End-to-end encryption with modern Forward Secrecy and Post-Quantum readiness
3. Anonymity at both content and metadata layers
4. No mandatory trusted operator
5. Easy self-hosting with low infrastructure cost
6. Polished mobile and desktop clients

## Mission

LibertyChat is a complete communication protocol and application stack designed to satisfy all six requirements. It treats every operator (server, relay, identity registry) as untrusted. Security and privacy are derived from cryptographic primitives and routing structure, not from the goodwill of any party.

## Guiding principles

### 1. No mandatory middleman
Any operator must be replaceable. Identity is held by the user (cryptographic keypair). Servers are dumb relays that can be self-hosted, public, or onion-routed depending on user preference. No server can be required to run the network.

### 2. Cryptographic guarantees over policy promises
Privacy properties are derived from math, not terms-of-service. Every claim ("server cannot read messages") must be technically enforceable, not policy-based.

### 3. Asynchronous-first
Messages must reach offline recipients reliably. The system must work when both parties are rarely online at the same time. Live calls are a feature on top, not the foundation.

### 4. Modular trust
Users choose their threat model per conversation. A casual chat with family can use direct connections; a sensitive conversation can route through onion mix-nets with extra latency. No one-size-fits-all anonymity.

### 5. Standards-aligned cryptography
Use audited, formally analyzed primitives. No custom crypto. Adopt MLS for groups, X25519 + Kyber hybrid for key exchange, Ed25519 + Dilithium hybrid for signatures.

### 6. Long-form mail as first-class citizen
Most messengers treat persistent long-form messages as second-class. LibertyChat provides a mail-style channel: long messages, threading, attachments, persistent storage on the user's mailbox server.

### 7. Federation without single points of failure
Servers can federate (talk to each other) but a server going down only affects its own users, not the network.

### 8. Open Source, AGPL
All code AGPL-3.0. Reproducible builds. External audits before any 1.0 release.

## Target users

- Self-hosters who want to run communication infrastructure for friends and family
- Communities (activist groups, journalists, technologists) needing strong anonymity guarantees
- Privacy-conscious individuals who want one tool for chat + voice + files + mail
- Developers who want a clean Rust reference implementation to build on

## Non-goals

- Mass-market consumer messenger competing with WhatsApp on user count
- Cryptocurrency or tokenization of any layer
- Identity verification through phone numbers, email, or government IDs
- Ad-supported or freemium model
- Lock-in to a single client or provider

## Inspiration and prior art

- **SimpleX** — asynchronous mailbox queues with no global user identifiers
- **Signal Protocol** — Double Ratchet and X3DH pioneered modern messaging crypto
- **Matrix / MLS** — federation model and standardized group encryption
- **Tor / Nym** — onion routing and mix-network designs for metadata privacy
- **Delta Chat** — using existing infrastructure (mail) as transport
- **Briar / Cwtch** — pure-P2P designs for high-threat scenarios
- **age** — modern, simple file encryption without PGP's pitfalls

LibertyChat aims to combine the strengths of these projects into one coherent stack rather than reinvent each layer in isolation.
