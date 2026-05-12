# Comparison with existing systems

This document compares LibertyChat's design choices to the most relevant existing projects, explaining what is borrowed, adapted, or rejected.

## SimpleX

LibertyChat takes the **mailbox-queue model** from SimpleX as its core. Per-contact unidirectional queues with no global user identity is the strongest privacy primitive available today and we adopt it directly.

### What we keep
- Asynchronous mailbox queues
- No global user identifier (per-contact pseudonyms)
- Out-of-band invitation links for connection setup
- Self-hostable single-binary server
- Multi-server distribution per user

### What we change
- **Crypto**: Replace SimpleX's custom protocol composition with standardized MLS (RFC 9420) for groups. SimpleX uses a Sesame-like extension that does not scale efficiently beyond ~50 group members.
- **Post-Quantum**: Hybrid Kyber + X25519 from day one. SimpleX is adding PQ but as a later evolution.
- **Multi-device**: First-class multi-device support via per-device sub-keys signed by master identity. SimpleX has a master-slave model with significant friction.
- **Mail layer**: Add a persistent long-form mail channel as a first-class feature. SimpleX has no comparable concept; messages are ephemeral chat-style.
- **Identity discovery**: Optional DNS-based discovery (`@user@domain`) so users with their own domain can be found without a centralized name service. SimpleX has no global discovery (privacy preserving but inconvenient).
- **Implementation language**: Rust instead of Haskell. Easier mobile FFI, smaller binaries, larger contributor pool, deterministic memory.

### What we explicitly reject
- SimpleX's tight coupling between SMP-server protocol and Haskell ecosystem
- Lack of a designed federation story between independent mailbox servers
- Custom group crypto (in favor of MLS)

## Signal

Signal Protocol's Double Ratchet is the gold standard for one-to-one E2EE. We use the same primitives.

### What we keep
- Double Ratchet for one-to-one conversations
- Sealed Sender concept (sender identity encrypted to server)
- Recently: PQXDH (post-quantum extension)

### What we change
- **No phone number requirement**. Identity is a keypair.
- **Self-hostable infrastructure**. Signal cannot be self-hosted in any practical sense.
- **No central server**. Federated mailbox model.
- **Mail layer**. Signal has no long-form persistent messages.
- **Decentralized identity discovery** (DNS-based, optional).

### What we explicitly reject
- Phone number as identity anchor
- Centralized infrastructure
- Closed ecosystem (Signal does not federate)

## Matrix / Element

Matrix's federation model is the most successful federated messenger today. We borrow lessons.

### What we keep
- Federation as a design choice (servers can talk to other servers)
- Standardized group crypto via MLS (Matrix is migrating to this)
- HTTP-based protocols where appropriate (easy to debug, well understood)

### What we change
- **Server complexity**: Matrix Synapse is heavy (Postgres, large RAM footprint, hours to set up). LibertyChat server is single binary with embedded storage, minutes to set up.
- **Identity model**: Matrix uses `@user:domain` which couples identity to a homeserver. LibertyChat keeps identity in the keypair; the domain is optional discovery only.
- **Metadata leaks**: Matrix homeservers know full social graphs of their users. LibertyChat's mailbox model prevents this structurally.
- **Resource use**: Matrix sync is bandwidth-heavy. LibertyChat uses pull-only retrieval from mailbox queues.

### What we explicitly reject
- Coupling identity to a specific server (homeserver lock-in)
- Olm/Megolm's complexity in favor of MLS standard
- The complexity-creep of Matrix specification in general

## Jami

Jami's pure-P2P model is conceptually elegant and works well for some use cases.

### What we keep
- Cryptographic identity as the only identifier
- Direct P2P connection for realtime media when possible
- No required server infrastructure for the user

### What we change
- **Asynchronous delivery**: Jami struggles when both parties are not online simultaneously. LibertyChat's mailbox layer solves this.
- **Mobile reliability**: Pure P2P is hard on mobile (battery, push notifications). The mailbox model integrates better with mobile constraints.
- **Discovery**: Jami uses OpenDHT plus a central name server (`ns.jami.net`). LibertyChat uses DNS-based discovery, which leverages existing infrastructure users already control (their own domain).

### What we explicitly reject
- Pure P2P for messaging (the asynchronous layer is critical)
- Reliance on DHT bootstrap nodes for everyday operation

## Session

Session built an interesting onion-routed messenger but made trade-offs we don't accept.

### What we keep
- Onion routing as an option for IP anonymity (we use Tor or Nym, not a custom network)

### What we change
- **Crypto**: Session removed Double Ratchet for compatibility with their async onion model, weakening Forward Secrecy. LibertyChat keeps Double Ratchet.
- **Self-host**: Session's Service Node network requires OXEN cryptocurrency stake to run a node. LibertyChat servers run on any commodity hardware with no economic gating.

### What we explicitly reject
- Cryptocurrency-based incentive model
- Removing Forward Secrecy in exchange for asynchronous routing
- Coupling network operation to a token economy

## Delta Chat

Delta Chat's "use existing infrastructure" approach is elegant for low-friction adoption.

### What we keep
- Idea of using existing infrastructure where possible (DNS for identity discovery, standard transport protocols)
- Bridging to email-like persistent messaging

### What we change
- **Don't use SMTP/IMAP as the transport**. Email infrastructure has too many privacy problems (server reads metadata, plaintext fallback for non-Autocrypt recipients, slow delivery, large attachment limits).
- **Real-time A/V**: Delta Chat delegates to external WebRTC services. LibertyChat has integrated A/V.

### What we explicitly reject
- Email as the message transport
- Dependency on Autocrypt (PGP-based) for encryption

## Tor / Nym

We integrate with Tor and Nym for IP anonymity rather than building our own onion network.

### What we use
- Tor as default optional onion-routing transport (mature, large network)
- Nym as alternative for users wanting mix-net traffic-analysis resistance

### Why we don't build our own
- Operating an onion network is a multi-decade engineering effort
- Existing networks have larger anonymity sets
- A purpose-built onion network for our messenger would be smaller and easier to attack
