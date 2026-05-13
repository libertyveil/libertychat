# Feature 5: Persistent Mail Layer

Step-by-step walkthrough of LibertyChat's unified message model. Mail and chat are not two separate features — they are two UI presentations of one single message type, with full cryptographic uniformity at the protocol layer.

This document supersedes earlier drafts that treated mail and chat as distinct subsystems.

---

## Step 1: The problem — chat and mail are two disconnected worlds today

Users juggle two unrelated tools with very different properties.

### Messenger (Signal, WhatsApp, Matrix, SimpleX)

- Ephemeral by design.
- No subject. No threading. Just a stream.
- No archival, no search across years.
- Attachments capped.
- Multi-device sync for long history is limited.

### E-mail (Gmail, Outlook, ProtonMail)

- Persistent, multi-year retention.
- Subject, threading, headers as first-class citizens.
- Attachments mostly unlimited (with provider quotas).
- Plaintext on servers (except provider-bound bubbles).
- **Plaintext fallback** when PGP-style encryption fails on a recipient → security collapses to the weakest link.
- Massive metadata leakage — `From`/`To`/`Subject`/`Date` never encrypted because SMTP routing requires them.
- Decades of retrofit hacks (SPF, DKIM, DMARC, MTA-STS, S/MIME, Autocrypt) trying to layer authentication onto a base protocol that never had it.

### What people actually need

Real conversations sit on a spectrum from short and ephemeral to long and persistent:

```
"ok"                                            → chat
"on my way down"                                → chat
"here's the address"                            → chat + attachment
"summary of today's meeting"                    → mail-style: subject, thread
"contract negotiation thread"                   → mail: persistent, searchable
"annual tax correspondence with the lawyer"     → mail: multi-year retention
```

Today users switch apps based on content. Conversations fracture across providers, each with its own crypto, its own identity model, its own search index. The same person is reached in two apps with two cryptographic identities.

### Nobody offers

A system that uses **the same identity, the same crypto, the same conversation, the same multi-device model** for the full spectrum — from "ok" to a years-long thread with attachments.

LibertyChat closes this gap by treating mail and chat as the **same operation**, distinguished only by the user's choice to attach a subject.

---

## Step 2: Foundational principles

Two architectural principles drive every decision in this layer:

### Principle 1 — Uniform maximum cryptography

Every message — a one-character chat ping, a multi-paragraph mail, a binary attachment chunk — flows through the **identical** cryptographic path:

- Hybrid PQ (X25519 + ML-KEM-768, Ed25519 + ML-DSA-65)
- MLS framing (RFC 9420) with Forward Secrecy + Post-Compromise Security
- Sealed Sender
- Fixed-bucket padding (1 / 4 / 16 / 64 / 256 KiB)
- VRF-rotated queue IDs
- Hardware-backed local storage where available

There is **no "chat mode" with lighter crypto**. A short chat message can be more sensitive than a long mail. The system cannot guess content importance, so it always treats every message at the maximum protection level.

### Principle 2 — Standards-native crypto, no retrofit hacks

LibertyChat refuses the email-evolution pattern: a flawed insecure base covered in decades of bolt-on authentication and encryption attempts. The rule is that every layer is cryptographically authenticated and encrypted **by design from the first byte**.

Concrete consequences:

- **No SMTP/IMAP bridge** to legacy email — not as a feature, not as an option, not in the future. Bridging would inherit every email weakness.
- **No optional encryption**. Encryption is non-negotiable; no plaintext fallback ever.
- **No "trust this IP/domain" assumption**. Authentication is always via signed cryptographic identity.
- **No silent downgrade** on negotiation failure. If the maximum crypto path cannot be established, the operation fails openly.
- **No external policy layer** to fix protocol problems. The protocol itself enforces correctness.

Where classical email needs SPF, DKIM, DMARC, MTA-STS, S/MIME, etc. to retroactively secure SMTP, LibertyChat has none of these — because identity, integrity, and transport security are intrinsic to the protocol, not bolted on.

---

## Step 3: One message — the unified data model

There is no `ChatMessage` and no `MailMessage`. There is **one** structure:

```rust
struct Message {
    message_id: MessageId,             // BLAKE3(sender_pubkey || timestamp || body || in_reply_to)
    sender_device: LeafIndex,          // sender's MLS leaf in the conversation group
    timestamp: u64,
    subject: Option<String>,           // optional, immutable once sent
    in_reply_to: Option<MessageId>,    // optional, points to direct parent
    body: MessageBody,                 // content or tombstone marker
    attachments: Vec<XftpRef>,         // references to chunked encrypted file blobs
}

enum MessageBody {
    Content(RichText),                 // normal payload
    Tombstoned {                       // deletion placeholder (Step 6)
        tombstoned_at: u64,
        tombstoned_by: LeafIndex,      // must equal original sender's device
    },
}
```

That is the entire wire-level vocabulary. There is no `Topic` struct, no `topic_id`, no `references` list, no `thread_id`, no mode flag.

### How the structure produces both chat and mail behavior

| Case                              | `subject`            | `in_reply_to`        | UI presentation                        |
|-----------------------------------|----------------------|----------------------|----------------------------------------|
| Quick chat message                | `None`               | `None`               | Stream view, free-floating             |
| Reply to a chat message           | `None`               | `Some(parent_id)`    | Stream view, displayed as a reply      |
| New mail / start of a thread      | `Some("Vertrag Q3")` | `None`               | Inbox entry, thread root               |
| Reply within a thread             | `None`               | `Some(parent_id)`    | Inside the thread tree                 |
| Sub-thread spawned from a chat    | `Some("Steuer")`     | `Some(chat_msg_id)`  | New inbox entry, lineage retained      |

### Subject inheritance

When the client needs the "effective subject" of a message (for display, search, threading), it walks the in_reply_to chain upward until it finds a message with a subject, or until the chain ends:

```
fn effective_subject(msg) -> Option<String> {
    if msg.subject.is_some() { return msg.subject.clone(); }
    match msg.in_reply_to {
        None         => None,
        Some(parent) => effective_subject(parent),
    }
}
```

A thread's identity is implicitly the `message_id` of the highest ancestor with a subject. There is no separate identifier.

### Message-ID is content-hashed

`message_id = BLAKE3(sender_pubkey || timestamp || body || in_reply_to)`.

This means a sender claiming `in_reply_to = X` must know `X`'s content to produce a valid `X` reference — the message-id is collision-resistant on content. Combined with sender signing (covered by MLS framing), the threading graph is **cryptographically authenticated** end to end. An attacker cannot fabricate or inject replies into a thread without legitimately participating in it.

---

## Step 4: Thread integrity as a structural invariant

A reply cannot exist without its parent. This is a hard rule, not a best-effort behavior.

### Consequences for sync and delivery

- **Mailbox queue** is acked per message. A receiver does not consider a message "consumed" until it is decrypted and stored locally. Lost messages are re-requested.
- **Out-of-order delivery** is tolerated transiently — the client buffers and applies messages in causal order using `in_reply_to` and timestamp.
- **New device joining a conversation** receives full historical message transcripts via the device-pairing channel before being considered active. There is no "from now on only" join — that would produce orphan replies (see [Feature 1, Step 10](01-unified-mls.md)).

### Consequences for deletion

Two distinct operations exist:

#### 4.1 Destructive delete (tombstone)

- **Granularity**: a single message — never a whole thread at once.
- **Who is allowed**: only the original sender of that specific message.
- **Mechanism**: the sender's device issues a signed delete request inside the MLS group; participating clients replace the message body with a `Tombstoned` marker.
- **What is preserved**: `message_id`, `subject` (if any), `in_reply_to`, timestamp, sender identity. The structure of the thread is intact; only the content is gone.
- **What is removed**: `body` content (text + attachment references). The attachment chunks on XFTP servers become unreferenced and are eventually garbage-collected by their retention policy.
- **No cascade**: deleting a parent does **not** delete its children. Other participants' replies remain — nobody can delete content they did not author, directly or indirectly.

#### 4.2 Local hide / archive

- **Granularity**: a whole thread, or individual messages, by client choice.
- **Who is allowed**: any participant, on their own devices, for themselves.
- **Mechanism**: per-user CRDT state (see Feature 16) marks the thread or message as hidden.
- **What is changed globally**: nothing. Other participants are unaffected.
- **Reversible**: hidden threads can be unhidden, found via search, etc.

#### Why no global thread-delete

Nobody can delete the content of others. A "delete thread for everyone" operation would let one participant erase the contributions of all others. Even with consent from all participants, this would weaken the invariant that authored content is owned by its author.

If a user wants a thread to disappear from their own view, that is a local hide. If a user wants to remove their own contributions from a thread, that is N individual tombstones — one per message they sent.

---

## Step 5: UI presentations — stream view vs inbox view

Two UI views render the same underlying message store, differently:

### Stream view (chat presentation)

- Chronological list of every message in the conversation
- Messages with a subject are visually highlighted as "thread starters" inline
- Reply chains are shown via indentation or quote-blocks
- Typing indicators, read receipts, reactions, message editing windows are enabled by default
- Notification urgency is high (per-message, immediate)
- Default retention is short (configurable per conversation; the user's preference)

### Inbox view (mail presentation)

- Lists only messages where `effective_subject != None` → only thread roots and threads they head
- Each entry shows: subject, latest activity, reply count, participant set
- Drill-in shows the thread as a tree (parent-child structure)
- Typing indicators and reactions are disabled by default
- Notification urgency is lower (batched, less aggressive)
- Default retention is long

Both views read from the same data. A user can switch freely. A message authored without a subject lives only in the stream view; a message with a subject appears in both.

### UI defaults are user-overridable per thread

Every UI default is **a default**, not a constraint. A user can:

- Pin a chat-style conversation to permanent retention.
- Enable typing indicators on a specific mail thread.
- Turn off read receipts for a specific contact.

The UI behavior never affects cryptographic protection — both views, all settings, route through the same maximum crypto stack.

---

## Step 6: Per-user state lives in CRDT, not in messages

State that is **about the user's relationship to a message** (rather than about the message itself) lives in a per-user-device CRDT (see Feature 16):

- Read / unread cursor per thread
- Starred / important flags
- Archived flag
- User-defined labels
- Personal title override for a thread (local renaming without changing the original subject)
- Retention override (per thread or per message)

This state syncs between the user's own devices via the device-pairing channel. Other participants in the conversation see none of it.

Why this matters: it means the message itself is immutable on the wire (signed and committed in MLS), while user-perception state can evolve freely. There is no "everyone sees Alice marked this read" leakage; her read cursor is hers.

---

## Step 7: Attachments via XFTP

Mail-style attachments (potentially large, potentially many) integrate uniformly:

- Files are split into 64 MiB chunks.
- Each chunk encrypted independently with a chunk key derived from a master file key.
- Chunks uploaded to one or more XFTP servers (independent from mailbox servers).
- The XFTP server sees only opaque encrypted chunks with random IDs; no filename, no link between chunks, no relation to senders or recipients.
- An `XftpRef` (master key + chunk list with server endpoints + chunk IDs) is embedded in the `Message`, inside the MLS-encrypted payload.

Consequences:

- **No effective size limit** — only XFTP-server quotas (user-configurable).
- **Multi-server distribution** — chunks of one file can be spread across several XFTP servers; no single server has a full view even of the encrypted form.
- **Streaming decryption** — clients can fetch chunks lazily for video playback or partial reads.
- **Resumable transfers** — interrupted uploads/downloads resume from the last successfully transferred chunk.

When a message is tombstoned, its `XftpRef` is removed; the chunks become unreferenced and are GC'd by the XFTP retention policy.

---

## Step 8: What the server sees

The mailbox server holds encrypted blobs in a queue. It does not see:

- `subject`
- `in_reply_to` / threading structure
- `body` content
- Attachment filenames, sizes (only padded buckets), MIME types
- Whether a message is a chat or a thread
- Whether a message is a tombstone
- The conversation graph between users

It does see:

- That an encrypted blob of bucket size N was uploaded
- That a recipient device (by ephemeral queue identity) ack'd it
- Aggregate storage and bandwidth usage per queue

The XFTP server analogously sees only:

- Opaque chunk uploads
- Opaque chunk downloads
- Storage usage per pseudonym

Identity, content, structure, threading — all of this is inside the MLS application payload and the XFTP chunk encryption layer.

---

## Step 9: What LibertyChat deliberately does not have

Some features common in classical email or some messengers are absent **by design**, not by accident.

| Feature                                | Why absent in LibertyChat                                                                                       |
|----------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| SMTP / IMAP bridge to legacy email     | Bridging inherits every email privacy and authentication weakness. Hard boundary.                              |
| Subject mutation / thread rename       | Subject is immutable. Local title override is a per-user CRDT (Step 6).                                         |
| `References` ancestor chain in message | Redundant. Walking `in_reply_to` is sufficient; thread integrity is enforced structurally (Step 4).             |
| CC / BCC                                | MLS group membership is the authoritative recipient list. To include extras, add them to the group.            |
| Hidden recipients (BCC-equivalent)     | Incompatible with MLS group transparency. A user wanting a private copy uses a separate 1:1 conversation.       |
| Read-receipts on by default            | Privacy default: do not leak whether the recipient has read a message. Per-thread opt-in available.            |
| Global thread-delete                   | One participant cannot erase others' authored content. Tombstoning is sender-only and per-message (Step 4.1).   |
| Subject-based thread auto-merge        | Classical email behavior is fragile and a frequent source of misthreading. Threads are joined by `in_reply_to` only. |
| Plaintext fallback                     | If the hybrid PQ + MLS path cannot be established, the operation fails openly. No degraded modes.              |

---

## Step 10: Why this is superior

- **Signal / WhatsApp**: chat only. No subject, no threading, no mail-style longform layer. Their model cannot represent multi-week formal correspondence in the same conversation as quick chats.
- **iMessage**: same — one continuous stream per contact, no thread concept.
- **Matrix**: rooms are persistent, but threading is a relatively recent addition and not deeply integrated. No mail-style longform UX.
- **Delta Chat**: uses real SMTP/IMAP under the hood — inherits every email privacy problem, requires Autocrypt for any encryption, plaintext fallback to non-Autocrypt recipients.
- **ProtonMail**: mail only, no realtime chat, closed end-to-end bubble (only encrypted between ProtonMail users).
- **Gmail / Outlook**: plaintext on the server, full metadata leakage, decades of retrofit hacks.
- **LibertyChat**: one message model, one cryptographic stack at maximum strength, one identity, one conversation. Chat and mail are two views over the same data. Thread integrity is a structural invariant. The server holds nothing usable. No retrofit hacks because every layer is authenticated and encrypted from the first byte.

---

This is the unified model. Mail is not a separate subsystem; it is a UI presentation of messages that have a subject. Chat is the UI presentation of messages that do not. The underlying protocol, crypto, identity, and storage are identical for both.
