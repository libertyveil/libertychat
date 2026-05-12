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

## The core thesis: infrastructure is corruptible

Even when operators are honest, competent, and well-intentioned, **any infrastructure that holds meaningful data is a target**. This is the foundational threat the project addresses.

Operators face pressures that no good intent can neutralize:

- **Legal coercion**: subpoenas, gag orders, national security letters, "voluntary cooperation" requests. The operator may have to comply even when it contradicts their stated values. Some jurisdictions criminalize refusal.
- **Compromise**: servers get hacked. Insider threats exist. Cloud providers can be coerced one level higher.
- **Drift over time**: today's privacy-friendly operator can be acquired, change leadership, change policies. The user's data outlives the operator's promises.
- **Partial corruption**: even when most nodes in a distributed system are honest, **a minority of malicious nodes can compromise large fractions of the network**. eMule's "spy nodes" demonstrated this at scale; modern onion networks face the same risk with Sybil attacks; Service-Node networks face it via stake concentration.
- **State-level adversaries**: with enough resources, even highly decentralized systems can be observed, correlated, or infiltrated.

The honest-operator assumption is the weakest link in every privacy claim. **A system designed to require honest operators is a system that will eventually fail.**

## What LibertyChat does about it

The architectural principle: **the infrastructure must hold no usable data**. Not "the operator promises not to look" — the operator must be **technically incapable** of looking.

Every layer of the system is designed so that compromise of any infrastructure component reveals **nothing useful**:

- **Mailbox servers** hold only encrypted blobs and per-queue authentication keys. They cannot decrypt content, cannot link queues to user identities, and cannot correlate different conversations of the same user.
- **Identity is a cryptographic keypair on the user's device.** There is no central registry. There is no "account database" that can be subpoenaed.
- **Discovery is opt-in via DNS** on the user's own domain. If a user does not publish a discovery record, no party knows the user exists at the protocol level.
- **Routing decisions are user-controlled.** A conversation can be routed direct, through Tor, through a mix-net, or through a chain of mailbox-relay hops. The user picks per conversation.
- **Multi-server distribution** is the default. A user's conversations live across multiple independent mailbox servers, so no single server has a complete view.
- **Cryptographic primitives are hybrid post-quantum** so that captured ciphertexts remain useless even decades from now.
- **Metadata is minimized at the protocol level**: padded message sizes, sealed sender envelopes, ephemeral queue identifiers that rotate over time.

The goal is that **when an investigator, hacker, or coercive authority approaches the infrastructure, there is genuinely nothing useful to give them**. The honest operator is no longer a privacy assumption — they are simply unable to compromise users even if they wanted to (or were forced to).

This shifts the trust model: instead of trusting operators to behave well, the system makes operator behavior cryptographically irrelevant.

## Guiding principles

### 1. No mandatory middleman
Any operator must be replaceable. Identity is held by the user (cryptographic keypair). Servers are dumb relays that can be self-hosted, public, or onion-routed depending on user preference. No server can be required to run the network.

### 2. Cryptographic guarantees over policy promises
Privacy properties are derived from math, not terms-of-service. Every claim ("server cannot read messages") must be technically enforceable, not policy-based. **The operator must not be in a position to break the user's privacy even when forced to.**

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
