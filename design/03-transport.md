# Transport layer

## Goals

- Reliable, low-latency delivery between client and mailbox server
- Resistance to traffic analysis when desired (onion routing optional)
- Mobile-friendly (handles network changes, poor connectivity)
- Standards-based, debuggable

## Wire transport

### QUIC over UDP

Primary transport is **QUIC** (RFC 9000):

- Built-in TLS 1.3
- 0-RTT or 1-RTT handshake (faster than TCP+TLS)
- Multiplexed streams without head-of-line blocking
- Connection migration (survives network changes — Wi-Fi to mobile data)
- UDP-based, escapes some HTTP-only firewalls

Rust implementation: `quinn` library.

### TLS 1.3 over TCP fallback

When UDP is blocked (corporate firewalls, restrictive networks):

- Standard TLS 1.3 over TCP port 443
- Same protocol semantics, slower handshake
- Detected automatically; client tries QUIC first, falls back

### Trust model

Mailbox server certificates are **self-signed** and **fingerprint-pinned** in the connection URL:

```
lvm://<base32-fingerprint>@<host>:<port>
```

The client verifies the server's certificate fingerprint matches the one in the URL. Standard CA chain validation is not used. This avoids dependency on the WebPKI certificate authority cartel and removes a class of attacks.

This matches the SimpleX approach. Self-hosters do not need to obtain Let's Encrypt certificates.

## Connection multiplexing

A single QUIC connection per (client, server) pair can carry:

- Multiple queue subscriptions (poll/notify on each)
- Outbound message writes
- Receipts and acks
- Server-pushed notifications

Streams are independent; one slow file upload doesn't block other operations.

## NAT traversal (for peer-to-peer media)

For real-time media (audio/video calls), the protocol must establish a direct P2P connection when possible.

We use the libp2p NAT traversal stack:

- **STUN** — discover public IP+port
- **ICE** — try all candidate pairs
- **Hole-punching** — coordinate simultaneous outbound packets to traverse symmetric NATs
- **TURN fallback** — relay when direct connection fails

Reference implementation: `libp2p` crate suite, or direct WebRTC ICE library.

## Optional onion routing

Users can enable onion-routed transport per conversation or globally. Two backends supported:

### Tor integration
- Built-in Arti (`arti-client` Rust library — the new Rust Tor implementation)
- Routes mailbox traffic through 3 Tor hops
- Server sees a Tor exit node IP, not the user's
- Higher latency (~200-500ms added)
- Best anonymity-set: Tor has millions of users

### Nym integration
- Mix-net with cover traffic and per-message timing randomization
- Stronger against global passive adversaries
- Requires NYM-token-based payments for relay capacity (or community gateway)
- Smaller anonymity set than Tor today, but better timing-attack resistance

Choice is per-user. Both can coexist; user picks per conversation or globally.

## Push notifications

A unique privacy challenge: mobile push (APNs, FCM) requires an external service to wake the app.

Approach:

1. **Run a LibertyChat-specific push relay** (analogous to SimpleX's notification service).
2. **Push payload contains no identifying content** — only an opaque token telling the device "you have a new message". Device wakes up, polls its mailbox.
3. **Push relay sees**: device tokens, frequency. Nothing else.
4. **Optional: cuckoo-filter-based pull mode** — device polls a digest of "any messages?" without revealing which.

Self-hosters can run their own push relay (and they should, for full privacy). Users without a push relay get email-style "wake on next foreground" behavior.

## Message size and batching

- Maximum single message: 64 KiB (text and small payloads)
- Larger payloads (files, images) chunked through the file storage layer
- Batched delivery: server can deliver multiple queued messages in one round-trip

## Reliability

- All client-server messages require acknowledgement
- On disconnect, queued messages remain on server until ack
- Server retains messages for configurable period (default 30 days, like SMTP)
- Delivery receipts are end-to-end encrypted between users (server doesn't see them)
