# MLS Protocol Walkthrough

Step-by-step walkthrough of how MLS (RFC 9420) operates within LibertyChat. Each step explains what happens and notes what makes this approach superior to the existing market.

LibertyChat applies MLS uniformly to **all** conversations — 1:1 chats are modeled as a 2-user MLS group whose members are all participating devices of both users. The same mechanics described below cover groups of size 2 and groups of thousands.

---

## Step 1: Creating a group

When Alice creates a new group, her client locally constructs an initial **group state**:

- A randomly-generated **group ID** (32 bytes, opaque, unique per group)
- An initial **epoch key** (the first encryption key for messages sent in this group)
- A **member tree** containing only Alice as the sole member

Nothing leaves Alice's device at this point. The group exists only in her local state — no server has any record of it yet.

### Why this is superior

- **Signal / WhatsApp**: groups get a server-assigned ID, registered against the user's phone-number account. The server knows the group exists from creation.
- **Matrix**: groups (rooms) are created on the user's homeserver, fully visible to that server from creation.
- **SimpleX**: groups exist as a coordinated set of 1-to-1 connections; the creator's server learns about each pair.
- **LibertyChat**: group creation is **entirely local**. No server learns the group exists until a message is sent. Creation is unobservable from outside the creator's device.

---

## Step 2: KeyPackages — pre-published invitation material

Before Alice can invite Bob, she needs a **KeyPackage** from him. A KeyPackage is a small signed bundle that Bob's device publishes in advance, containing:

- A **public encryption key** (HPKE, hybrid post-quantum) for receiving Welcome messages
- Bob's **identity public key** (for authentication)
- Supported cryptographic ciphersuite parameters
- A signature by Bob's identity key over the bundle

Each of Bob's devices publishes its own KeyPackage. They are stored at a location chosen by Bob (his mailbox server, his DNS-based identity descriptor, or both for redundancy).

When Alice wants to invite Bob, she fetches one fresh KeyPackage per Bob device. **Bob's devices do not need to be online** at that moment — the KeyPackages are already published.

### Why this is superior

- **Signal**: prekeys exist (similar concept) but only on Signal's centralized prekey server. Cannot be self-hosted.
- **WhatsApp**: prekeys hosted on Meta's servers, no user control.
- **Matrix**: device keys are exposed through the homeserver. Inviting requires Bob's homeserver to be reachable.
- **SimpleX**: no prekey concept; both parties must complete a key exchange in the opening message round-trip, meaning the inviter cannot prepare the invite asynchronously.
- **LibertyChat**: KeyPackages can be published anywhere the user controls — own mailbox server, DNS-based identity descriptor, even mirrored across multiple locations. The publishing layer is interchangeable; cryptographically only the bundle's signature matters.

---

## Step 3: Alice invites Bob — Add proposal, Commit, Welcome

Alice now adds Bob to the group. Three MLS objects are produced in sequence, entirely on Alice's device:

1. **Add proposal** — an MLS structure naming Bob's KeyPackage as the new member to be added. The proposal contains no secret material; it is the "intent to add" record.
2. **Commit** — an MLS message that bundles one or more proposals and applies them, advancing the group from **epoch N** to **epoch N+1**. The Commit derives a new epoch key, updates the TreeKEM ratchet tree, and produces a fresh group secret that only current members (and the new member) can recover.
3. **Welcome message** — a separately-encrypted package addressed specifically to Bob's KeyPackage. The Welcome contains exactly the group state Bob needs to join: the ratchet tree, the current epoch's secrets, the group ID, and the membership list. It is encrypted under the HPKE public key from Bob's KeyPackage, so only the Bob device that owns the matching private key can open it.

Alice uploads the Welcome to Bob's mailbox queue and uploads the Commit to the group's own mailbox queue. Bob may be offline; both messages wait in their respective queues.

### Why this is superior

- **Signal / WhatsApp**: adding a member to a group requires the inviter to perform a separate X3DH handshake or sender-key distribution to each existing member individually, scaling linearly in cost.
- **Matrix (Megolm)**: a new member receives the current sender's outbound session, but historical messages from other senders are not retrievable; member changes require rotating per-sender keys ad hoc.
- **SimpleX**: each existing member must establish a new 1:1 connection with the joining member — N pairwise handshakes for a group of N. There is no "Welcome" abstraction.
- **LibertyChat**: a single Commit advances the entire group atomically, and a single Welcome bootstraps the new member. The number of cryptographic operations scales **logarithmically** in group size, not linearly.

---

## Step 4: Bob processes the Welcome and joins

Bob's device retrieves the Welcome from its mailbox queue. Processing the Welcome:

1. **Decrypt** the Welcome with the HPKE private key corresponding to the consumed KeyPackage (the KeyPackage is now retired — single-use).
2. **Validate** the embedded group state: signatures on the ratchet tree, identity assertions for existing members, the group ID, the current epoch number.
3. **Install** the group state locally: the ratchet tree, the current epoch key, the epoch authenticator, the membership view.
4. **Mark** the KeyPackage as consumed so it cannot be replayed to add Bob to a forged group.

Bob is now a full member of epoch N+1 and can immediately decrypt any subsequent message encrypted to that epoch.

### Why this is superior

- **Signal / WhatsApp**: a newly added member never gets access to historical group state; they receive only future messages and must rebuild context manually.
- **Matrix**: history visibility is a server-controlled policy decision; the homeserver decides whether the new member can see past messages, and the encryption keys must be redistributed in-band.
- **SimpleX**: joining means N separate connection setups; nothing equivalent to a single bootstrap package.
- **LibertyChat**: one decryption operation gives Bob full cryptographic membership. KeyPackage consumption prevents replay. The group state is signed, so Bob can independently verify it without trusting any server.

---

## Step 5: Sending the first message

Alice sends a message in epoch N+1. The encryption flow:

1. Derive a per-message key from the current **epoch secret** plus a per-sender ratchet counter (the application-message ratchet). This advances forward after each message Alice sends.
2. Encrypt the plaintext with AEAD (ChaCha20-Poly1305 in the active ciphersuite).
3. Wrap the ciphertext in an MLS **PrivateMessage** structure carrying the epoch number, sender index, and a MAC over the framing.
4. Upload one copy to the group's mailbox queue. The mailbox server fans the encrypted blob out to all subscribed devices (Alice's other devices + all of Bob's devices).

Each recipient device looks up the matching epoch secret, advances its receive-side ratchet for sender "Alice", and decrypts. Old per-message keys are deleted immediately after decryption.

### Why this is superior

- **Signal Sender Keys**: each member maintains a separate sender chain, but adding/removing a member forces all senders to rotate. Cost grows with member count.
- **Matrix (Megolm)**: each sender has a megolm session; receivers store the session indefinitely and rotate slowly, weakening Forward Secrecy at the per-message level.
- **SimpleX**: a group message is sent N-1 times, once per pairwise connection. Bandwidth scales linearly with group size.
- **LibertyChat**: one upload, server-side fanout, per-message forward-secret key deleted on use. Same code path for a 2-person 1:1 and a 2000-person group.

---

## Step 6: Adding a third member (Carol) — TreeKEM scaling

Alice (or any member with permission) adds Carol the same way Bob was added: Add proposal, Commit, Welcome. The interesting property is the **cost of the Commit**.

MLS organizes members as leaves in a left-balanced binary tree. Each non-leaf node holds an HPKE keypair derived from its descendants. A Commit only needs to update the keys along the path from the changed leaf to the root — **O(log N)** keys for a group of N members.

For Carol joining a 1024-member group:

- Update affects ~10 internal nodes (log2 1024)
- Commit message is small (a few hundred bytes plus path encryption material)
- All existing members process the Commit in roughly the same constant per-update cost

### Why this is superior

- **Signal Sender Keys / WhatsApp**: adding a member requires every existing member to issue a fresh sender key to the new member, **O(N)** messages.
- **Matrix (Megolm)**: each sender re-establishes their megolm session as needed; the operation is amortized but the protocol does not bound the cost.
- **SimpleX**: each existing member must complete a fresh 1:1 handshake with Carol — **O(N)** handshakes, each multi-round-trip.
- **LibertyChat**: **logarithmic** cost is what makes thousand-member encrypted groups feasible. This is the structural advantage MLS was designed to deliver, and we get it for 1:1 chats as well.

---

## Step 7: Removing a member — Forward Secrecy

Alice removes Carol from the group. The operation:

1. **Remove proposal** — names Carol's leaf index for removal.
2. **Commit** — advances the group to a new epoch. The Commit re-derives all keys on the path from Carol's old leaf to the root using fresh randomness contributed by the committer. Carol's leaf is marked blank (or replaced by padding).
3. The new epoch key is computable only from the updated tree, which Carol no longer has access to.

From the moment the Commit is processed, Carol **cannot decrypt any future message** sent in the group. Even if Carol kept her old device keys, the new epoch's secrets are derived from fresh material she has no path to.

Historical messages remain readable by all remaining members (no retroactive secrecy from co-members — by design, since they already saw the messages).

### Why this is superior

- **Signal Sender Keys / WhatsApp**: removing a member forces every remaining member to rotate their sender keys. The protocol assumes cooperation; a member who doesn't rotate continues to leak future messages to the removed party until they do.
- **Matrix (Megolm)**: removing a member triggers a megolm session rotation request, but enforcement is policy-based and clients can be slow or forgetful, leaving a window during which the removed member still has the active session.
- **SimpleX**: there is no built-in "group remove" — the removing party would need to convince all other members to drop their 1:1 connection to the removed member. Coordination is application-level.
- **LibertyChat**: removal is a **single cryptographic operation** that immediately invalidates the removed member's access for everyone. No cooperation required from other members; no policy enforcement gap.

---

## Step 8: Epoch transitions and the message ratchet

Two ratchets operate in MLS:

1. **Epoch ratchet** — advances every time a Commit is processed (member add, member remove, key update, periodic refresh). Each new epoch has a fresh epoch secret derived from the previous epoch and the Commit's contributed entropy.
2. **Application-message ratchet** — within a single epoch, each sender's stream of messages advances a per-sender symmetric ratchet. Per-message keys are derived in order and deleted after use.

Old epoch keys are deleted on epoch transition. Old per-message keys are deleted on use.

A device that is compromised at time T can decrypt:

- Messages it has not yet consumed from its current queue (small window)

It **cannot** decrypt:

- Any past message whose per-message key has been deleted
- Any past message from an earlier epoch (epoch secrets gone)

### Why this is superior

- **Signal Double Ratchet**: provides Forward Secrecy per message, but only for 1:1. The group equivalent (Sender Keys) does not have a per-message ratchet — sender keys are rotated coarsely.
- **WhatsApp**: same as Signal Sender Keys — coarse rotation, weaker per-message FS in groups.
- **Matrix Megolm**: sender keys ratchet but slowly (configurable); the protocol explicitly trades FS for performance.
- **SimpleX**: Double Ratchet per 1:1 connection, but a group conversation = N connections, so a compromise of any member affects N pairwise streams.
- **LibertyChat**: **per-message Forward Secrecy in groups of any size**, plus epoch-level FS on every membership change. Both ratchets are standardized; no per-vendor weakening.

---

## Step 9: Post-Compromise Security

Forward Secrecy protects **past** messages from future compromise. **Post-Compromise Security (PCS)** is the dual: it protects **future** messages from a past compromise that has since been "healed".

MLS provides PCS through key-update Commits. Any member can at any time issue an **Update proposal**: it replaces their leaf's keypair with a fresh one and commits, deriving new path keys up to the root.

After the Update Commit is processed:

- An attacker who previously stole that member's private keys can no longer compute the current epoch secret
- New messages are encrypted under keys derived from the fresh material, which the attacker has no path to

In LibertyChat, clients perform an automatic Update Commit on a configurable schedule (e.g. every 24 hours of active group participation, or after each app launch) so that PCS is recovered without user intervention.

### Why this is superior

- **Signal Double Ratchet**: provides PCS through DH ratcheting in 1:1, but **only when both parties exchange messages**. A silent conversation never heals. Groups (Sender Keys) have no PCS at all.
- **WhatsApp**: same limitation; Sender Keys in groups have no PCS mechanism.
- **Matrix Megolm**: no PCS — once a megolm session is compromised, all future messages in that session are compromised until the session is manually rotated.
- **SimpleX**: PCS per 1:1 ratchet (good); groups inherit the weakness of any one compromised pairwise connection.
- **LibertyChat**: **active, scheduled PCS for all conversations**, regardless of whether other members are sending. The Update Commit mechanism is part of the standardized protocol, not a vendor extension.

---

## Step 10: Multi-device — every device is an MLS member

LibertyChat treats each of a user's devices as a **first-class MLS member**. There is no "primary device" and no shared device key.

When Alice has a phone and a laptop:

- Both devices publish their own KeyPackages (separately signed by Alice's identity key).
- When Bob invites Alice, Bob fetches one KeyPackage per Alice device and adds them all as separate leaves to the MLS group.
- Each Alice device holds its own MLS state, ratchets independently, and decrypts messages independently from the same mailbox-queue fanout.

Adding a new device later (Alice buys a tablet):

- The new device publishes a KeyPackage.
- Any of Alice's existing devices (which is a current group member) issues an Add proposal + Commit + Welcome targeted at the new device.
- **Full historical message transcripts are mandatory** — an existing device re-encrypts the conversation history to the new device via a dedicated device-pairing channel (itself an MLS group between Alice's devices). The new device is not considered "joined" until this transfer completes.
- Rationale: LibertyChat enforces **thread integrity as a structural invariant**. A member cannot hold a reply without also holding its parent. Allowing a new device to start "from now on only" would create reply chains pointing at unknown ancestors, breaking the invariant.
- Trade-off: the blast radius of a compromised newly-provisioned device is larger (it gets historical content too). This is accepted in exchange for uniform conversation state across all of a user's devices.

Removing a device (Alice loses her phone):

- Any other Alice device issues a Remove proposal + Commit for the lost device's leaf.
- The lost device immediately loses access to all conversations Alice was in.

### Why this is superior

- **Signal**: a "linked device" shares a derived key with the primary; the primary device must approve linking. Lost-phone recovery is awkward; there is no protocol-level "revoke this device from all groups" — it requires de-registering at the server.
- **WhatsApp**: multi-device added late and works through a complex companion-mode protocol with limited group support.
- **Matrix**: each device has its own keys, but device verification and cross-signing are layered on top of Olm/Megolm with substantial UX friction; revoking a device leaves a long tail of un-rotated sessions.
- **SimpleX**: has a primary-device model with experimental secondary-device support that requires the primary to be online; not a true peer multi-device.
- **LibertyChat**: every device is a peer MLS member; multi-device is **the same primitive as multi-user**. Device add/remove is logarithmic-cost and immediately effective across **all** the user's conversations. No primary-secondary asymmetry, no central registry to update.

---

## Step 11: Per-message authentication and sender identification

Every MLS application message is authenticated in two layers:

1. **Symmetric MAC** over the message framing, using a key derived from the epoch secret. Proves the message came from a current group member without revealing which one to anyone outside the group.
2. **Sender index** in the framing identifies which leaf (i.e. which device) sent the message. The recipient verifies that the sender's leaf is currently occupied by a known device with a known identity key. The identity-key-to-device binding was established at member-add time and signed.

Combined, the recipient knows:

- The message came from a current member of the group (MAC verifies under the shared epoch key)
- The sender is specifically the device at leaf index K (sender index in the framing)
- That device belongs to user X (signed identity binding from the Add)

A device that is not a current member cannot produce a valid MAC. A device that is a current member but tries to forge another member's sender index cannot, because the application-message ratchet keys are per-leaf.

### Why this is superior

- **Signal / WhatsApp**: per-message authentication exists but is bound to the sender's identity key. There is no concept of "sender as a device within a group" — group authentication is best-effort and depends on sender-key trust assumptions.
- **Matrix (Megolm)**: messages are signed by the sending megolm session, but the binding from session to device to user requires cross-signing infrastructure that is famously fragile.
- **SimpleX**: messages are authenticated per pairwise connection. Within a group, each message arrives over a single pairwise connection, so the recipient knows who sent it — but the group membership view itself is not authenticated by a common protocol object.
- **LibertyChat**: **standardized, cryptographically enforced sender identification** at the group level. A member cannot be impersonated by another member, and a non-member cannot inject messages even with full network access.

---

## Summary in one paragraph

LibertyChat uses MLS (RFC 9420) uniformly for every conversation. Each device publishes a KeyPackage so it can be added asynchronously. Groups are created locally with no server knowledge. Members are added via Add+Commit+Welcome; removed via Remove+Commit. Membership operations cost **O(log N)** via TreeKEM. Every Commit advances the epoch and rotates keys (Forward Secrecy). Scheduled Update Commits recover from compromise (Post-Compromise Security). Each user's devices are independent MLS members, so multi-device is the same primitive as multi-user. Per-message authentication and sender identification are standard parts of the framing. The same code path handles a 2-device 1:1 chat and a 2000-member group.

This is what no other deployed messenger currently does: **one standardized, formally analyzed group-encryption protocol applied to every conversation, with hybrid post-quantum primitives, logarithmic scaling, and first-class multi-device** — all on top of a self-hostable mailbox-queue transport that holds no usable data.
