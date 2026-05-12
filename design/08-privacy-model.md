# Privacy and threat model

## Adversaries considered

LibertyChat's design considers the following adversary types:

### 1. Casual observer
Coffee-shop Wi-Fi snooper, ISP doing DPI, generic network observer.

**Defense**: TLS 1.3 / QUIC encrypts all traffic. Observer sees connections to mailbox servers but no content.

### 2. Mailbox server operator
Person or entity running the user's mailbox server, possibly compromised or coerced.

**Defense**: All message content is E2EE. Server sees encrypted blobs and per-queue metadata only. No global identity, so server cannot correlate one user's queues. Multiple queues per user can live on different servers.

### 3. Compromised TURN/relay operator
Person or entity running a TURN server used during a call.

**Defense**: SRTP media is E2E encrypted between endpoints. Relay sees only encrypted bytes, IPs, and call timing.

### 4. Malicious peer (your contact)
Someone you've connected with who tries to gather info about you beyond what they should see.

**Defense**: Cryptographic identities cannot be impersonated. Per-conversation keys cannot be used to impersonate the user to others. If you used Tor, the peer doesn't see your IP.

Limit: a peer always knows what you've sent them. No protocol can prevent that.

### 5. Global passive adversary
State-level actor observing large fractions of the internet (NSA, GCHQ).

**Defense (partial)**: Onion routing (Tor or Nym) hides traffic patterns. Multiple-mailbox-server distribution prevents single-point correlation. With cover traffic (Nym), timing analysis is harder.

Limit: traffic analysis at extreme scale is fundamentally hard to defend against. LibertyChat does not claim to defeat a state-level adversary observing all internet traffic. For that level, Briar/Cwtch with Tor + face-to-face key exchange remains the gold standard.

### 6. Compromised endpoint
Malware on the user's device with read access to memory and storage.

**Defense (none)**: All bets are off. Forward Secrecy bounds the damage to messages from after compromise; older messages with deleted ratchet keys remain encrypted on disk and undecryptable. But an attacker with live device access reads anything.

### 7. Coerced user
User is forced to reveal credentials.

**Defense (none for full coercion)**:
- Optional **duress passphrase**: a fake passphrase that opens an empty profile (planned feature).
- Optional **plausible deniability**: hidden profiles that don't appear without the right passphrase.

Limited: coercion is fundamentally outside the scope of cryptography.

### 8. Future quantum attacker
Quantum computer capable of breaking ECDH and EdDSA.

**Defense**: Hybrid post-quantum keys from day one. Even if classical primitives break, PQ component (Kyber, Dilithium) provides security.

Limit: quantum-safe schemes are newer, less battle-tested. Hybrid mode is intentional belt-and-suspenders.

## What metadata leaks

| Information | Who can see it | Mitigation |
|---|---|---|
| You connect to mailbox server X at time T | server X, ISP, ISP-level observer | Tor / Nym onion routing |
| Queue Q has a message arriving | server hosting Q | distribute queues across servers |
| Message size N bytes | server, network observer | padding to fixed bucket sizes |
| Two queues are touched by same client (same TLS connection) | server | use separate connections / Tor circuits per queue |
| Recipient is online | server (queues are subscribed) | minimize subscription frequency, use cover-poll |
| Call is happening between IP A and IP B | TURN server (if used), endpoints | use TURN-only mode; both endpoints behind Tor / VPN |
| User has identity IPK X | discovery server (if used opt-in) | don't publish discovery record |

## Per-conversation privacy choices

Users can set privacy levels per conversation:

| Level | Direct connection | Onion routing | Cover traffic | Use case |
|---|---|---|---|---|
| **Casual** | yes | no | no | Family chats |
| **Private** | yes | optional | no | Friends, colleagues |
| **High** | no (forced relay) | yes (Tor) | no | Sensitive topics |
| **Maximum** | no (forced relay) | yes (Nym) | yes | Activist coordination |

Higher levels add latency and bandwidth cost. User chooses per conversation.

## Defaults

Out of the box, LibertyChat defaults to:

- E2EE for everything (non-negotiable)
- Direct connections to mailbox servers (no Tor by default — performance reasons)
- Identity discovery off (must be enabled per user)
- Push notifications via LibertyChat-operated relay (privacy-preserving payload)

User can opt into stronger settings as desired.

## What we do not promise

- **Anonymity from the recipient**. If you talk to someone, they know it's you (your IPK). Use a separate identity if you need anonymity from a counterparty.
- **Resistance to traffic analysis at state scale**. We provide tools (Tor, Nym) but cannot rule out attacks by adversaries with global observation.
- **Protection against device compromise**. Standard limitation of any messenger.
- **Quantum security guarantees forever**. Hybrid PQ provides margin; we'll upgrade as PQ research progresses.
