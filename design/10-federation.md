# Federation

## What federation means here

A user on mailbox server A can communicate with a user on mailbox server B without either user changing servers. Servers are independent operators that interoperate via a documented protocol.

This is the email model and the Matrix model: distributed identity, federated infrastructure.

## Why federation matters

- **No single point of failure**: One server going down only affects its own users.
- **Operator diversity**: Different servers can have different policies, jurisdictions, retention periods.
- **User choice**: Users can pick a server they trust or run their own.
- **Resists capture**: No single entity can shut down the network.

## Federation in LibertyChat

Identity is **server-agnostic**. A user's identity is their cryptographic public key, not `@user@server`. So in a sense, LibertyChat is more decentralized than Matrix from the start: there's no "homeserver" coupling.

What federates is **routing**:

- Mailbox servers route messages by examining queue addresses
- A queue address points to a specific server
- When you write to someone's queue, your client connects to that server directly

In this sense, every server "federates" with every other server by virtue of the open protocol. There's no handshake or trust relationship needed between servers; they don't even communicate with each other directly.

## Server-to-server protocols

Some operations DO require server-to-server communication:

### Message forwarding

If your mailbox server is offline, you can configure a backup server. Messages destined for your queue can be forwarded by the primary to the backup automatically.

### Large file transfer

XFTP-style chunked file storage may use multiple servers for resilience. Chunks can be replicated across mailbox servers.

### Identity discovery cache

Discovery responses (DNS-based) can be cached and shared between servers in a federation network to reduce DNS lookup load. Requires mutual trust; servers within a "federation alliance" can pool caches.

## Federation alliance model

Optional layer on top of the basic protocol:

- A group of servers forms an alliance with a shared trust policy
- Alliance members publish their list of peers
- Alliance members share resource usage (e.g. push notification relays)
- Users can prefer alliance servers in their connections

This is a soft federation: not required, but useful for community-operated networks.

## Anti-abuse

Open federation invites abuse:

- Spam: massive sending of unsolicited connection requests
- DoS: flooding queues with junk messages
- Misuse: server operators serving illegal content

Mitigations:

### Spam prevention by structure
LibertyChat doesn't have an open delivery mechanism. To send to someone, you need an established connection. Discovery requires the recipient to accept first. Spam at the protocol level is structurally hard.

### Per-queue quotas
Mailbox servers enforce limits: max messages per queue per hour, max queue size. Excess writes are rejected.

### Operator policy
Each server operator decides their own policies (retention, file size limits, paid tiers if any). The protocol does not prescribe these.

### Reputation systems (optional)
Queue creators can choose to require reputation tokens for unsolicited connection requests. The reputation system itself is decentralized (proof-of-work or similar non-cryptocurrency mechanisms). Out of scope for v1.0.

## Cross-server group chats

For MLS groups spanning multiple mailbox servers:

- Group has a "primary queue" on one of the servers
- Members on other servers connect to the primary queue to read group messages
- MLS handles the cryptographic state regardless of which server hosts the queue
- Members can be on any server; the group queue is just a routing hub

This means a group of 100 members across 50 different mailbox servers works fine. Each member connects to whichever server hosts the group's queue.

## Federation policy

**Default**: open federation with anyone. Any LibertyChat client can connect to any LibertyChat server.

**Server-side**: operators can set policies:
- Whitelist mode (only specific source IPs / authenticated users)
- Reject mode (block specific IP ranges)
- Geographic restrictions

**Client-side**: users can choose to send/receive only via certain server lists. The protocol allows it, the operator might enforce it.

## Differences from email federation

| Aspect | Email | LibertyChat |
|---|---|---|
| Identity | `name@domain` (server-bound) | cryptographic key (server-agnostic) |
| Server-to-server | SMTP between MX records | client-to-server only (no S2S routing required) |
| Spam | massive problem (millions of emails/day) | structurally limited (no open inbox) |
| Encryption | optional (PGP/S/MIME) | mandatory E2EE |
| Delivery guarantees | best-effort SMTP | acknowledged delivery via mailbox queues |
| Server operator visibility | sender, recipient, subject, body | nothing (E2EE) |

## Differences from Matrix federation

| Aspect | Matrix | LibertyChat |
|---|---|---|
| Identity | `@user:domain` (server-bound) | cryptographic key (server-agnostic) |
| Server-to-server | constant gossip via federation API | none required by default |
| Resource cost | high (servers replicate room state) | low (servers are dumb queues) |
| Server-to-server protocol | proprietary, complex | none in basic mode; optional alliance protocol |
| Federation barrier | run a homeserver | run a mailbox server (much simpler) |
