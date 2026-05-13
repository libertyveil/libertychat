# Feature 5: Persistent Mail Layer

Step-by-step walkthrough of LibertyChat's persistent mail layer — long-form, threaded, attachment-friendly messages alongside ephemeral chat, in the same application and the same cryptographic stack.

---

## Step 1: The problem — messenger and mail are two disconnected worlds today

Users currently juggle two unrelated tools with very different properties:

### Messenger (Signal, WhatsApp, Matrix, SimpleX)

- **Ephemeral by design.** Messages are short, chat-style, often relevant only for seconds to hours.
- **No subject.** No threading structure. The user scrolls through a stream.
- **No "archive"** as a filing system. Search UX is rudimentary.
- **Multi-device sync for long history is limited.** Signal stores messages only locally; lost phone means lost history.
- **Attachments often capped** (Signal 100 MB, WhatsApp 2 GB, Matrix homeserver-dependent).

### E-mail (Gmail, Outlook, ProtonMail)

- **Persistent** with decades-long retention.
- **Subject, threading, and headers** as first-class citizens.
- **Attachments of essentially any size** (with provider limits).
- **Plaintext on the server** (except for ProtonMail's closed bubbles), full plaintext indexing by the provider.
- **Plaintext fallback** when attempting PGP — a recipient without a key receives the message unprotected.
- **Metadata massively leaked** — From/To/Subject/Date are never encrypted because SMTP routing requires them.

### What people actually need

A scale from **short and ephemeral** to **long and persistent**:

```
"ok"                                  → chat
"on my way down"                      → chat
"here's the address"                  → chat / short with attachment
"summary of today's meeting"          → mail-style: subject, thread, longer text
"contract negotiation thread"         → mail: persistent, searchable, attachments
"annual tax correspondence with the lawyer" → mail: multi-year retention
```

Today users switch apps based on content. The conversation fractures: Signal chat "hey, did you read the contract?" → Gmail mail with the contract → back to Signal "and?". Three privacy domains, three auth models, three search indexes, no end-to-end encryption across them.

### Nobody offers

An application that uses **the same identity, the same crypto stack, the same multi-device model** for the full scale — from "ok" to a years-long mail thread.

- **Signal / WhatsApp**: chat only, no mail concept.
- **Matrix**: rooms only, no mail model.
- **Delta Chat**: uses **actual SMTP/IMAP infrastructure** — inheriting every email privacy problem. Server sees headers, subject, routing metadata. Recipients without Autocrypt setup get plaintext.
- **ProtonMail**: mail-only, closed bubble, no realtime chat.

**LibertyChat closes this gap**: mail as a native second mode on the same stack, with the same privacy guarantees as chat.

---

## Step 2: How LibertyChat solves it — mail as a second mode on the same stack

Mail is **not a separate protocol** — it is a different usage form of the same stack.

### Two modes on one foundation

```
                Application Layer
   ┌─────────────────────────┬─────────────────────────┐
   │      Chat mode          │      Mail mode          │
   │  - short messages       │  - subject + body       │
   │  - no subject           │  - threading tree       │
   │  - linear stream        │  - folders / labels     │
   │  - ephemeral-friendly   │  - persistent default   │
   │  - read markers         │  - read/unread + flags  │
   └────────────┬────────────┴────────────┬────────────┘
                │                         │
                └────────┬────────────────┘
                         ▼
              ┌────────────────────────┐
              │   MLS conversation     │  ← identical for both
              │   (RFC 9420)           │
              └────────────┬───────────┘
                           ▼
              ┌────────────────────────┐
              │  Mailbox queue         │  ← identical for both
              │  Encrypted blobs       │
              └────────────────────────┘
```

**Both modes use the same primitives**:

- The same MLS group for the relationship with a contact (or group).
- The same mailbox queue for asynchronous delivery.
- The same identity keys, the same sealed-sender encryption.
- The same multi-device members (every device receives both chat and mail).

The only difference is the **application-layer structure** of the message.

### Concretely: what distinguishes a mail from a chat message

A chat message:

```rust
ChatMessage {
    sender_device: LeafIndex,
    timestamp: u64,
    body: String,            // typically 1-3 sentences
    reply_to: Option<MessageId>,
    attachments: Vec<XftpRef>,   // rare
}
```

A mail message:

```rust
MailMessage {
    sender_device: LeafIndex,
    timestamp: u64,
    subject: String,             // ← new
    body: RichText,              // Markdown / longer text
    thread_id: ThreadId,         // ← new, identifies the thread
    in_reply_to: Option<MessageId>,
    references: Vec<MessageId>,  // ← thread-tree parents
    attachments: Vec<XftpRef>,   // expected, large files fine
    flags: MailFlags,            // Read/Unread, Star, Important, Archived
    labels: Vec<Label>,          // ← user-defined labels
    retention_policy: Retention, // permanent / N-days / N-years
}
```

Both are encrypted as **MLS Application Messages** — same AEAD path, same Forward-Secrecy ratchet. The mailbox server sees an encrypted blob in either case; it does not know the mode.

### How the UI decides which mode

When composing a new message, the app decides based on:

- **Explicit user choice** ("new mail" vs. "quick message")
- **Content heuristic** (longer than N characters + contains paragraphs → suggest mail mode)
- **Recipient preference** (some contacts are tagged mail-only, some chat-only, some both)

The receiving app **automatically routes** based on the presence of `subject` — the message lands in the mail inbox or the chat stream. Both arrive over the same MLS group.

### What this means in practice

The user has **a single relationship** with Bob (one MLS group containing all devices on both sides). Inside that relationship flow:

- Short chat ("on my way")
- Longer mails with subject ("summary of today's meeting")
- Attachments of unlimited size via XFTP
- File drops
- Audio/video call signaling

**One conversation, every communication form.** No app switch, no identity switch, no encryption-domain switch.

### Server side

On the mailbox server there is no difference between chat and mail — both are blobs in the same queue. The only differences:

- **Retention policy per blob** (chat: 7 days default; mail: permanent default).
- **Storage quota for mail blobs counted separately** — a user may want 50 GB of mail archive but only 1 GB of chat backlog.

Both policies live in the encrypted metadata the server needs for storage accounting (but cannot decrypt for content).

---

## Step 3: Threading model — how LibertyChat structures mail threads

Email threading is classically a mess: every client interprets `In-Reply-To` and `References` headers differently. Some flat (Gmail), some tree-structured (mutt, Thunderbird), some subject-based (Outlook).

LibertyChat implements **strictly tree-structured threading** with cryptographically verifiable parent relationships.

### How a thread is built

Every mail message has a unique `MessageId`. When Bob replies to Alice's mail:

```rust
Alice's mail:
  message_id:   "msg_abc123"
  thread_id:    "msg_abc123"        // = own ID for thread roots
  in_reply_to:  None
  references:   []

Bob's reply:
  message_id:   "msg_def456"
  thread_id:    "msg_abc123"        // ← thread root
  in_reply_to:  "msg_abc123"        // ← direct parent
  references:   ["msg_abc123"]      // ← ancestors

Carol's reply to Bob:
  message_id:   "msg_ghi789"
  thread_id:    "msg_abc123"        // ← same thread root
  in_reply_to:  "msg_def456"        // ← Bob's mail
  references:   ["msg_abc123", "msg_def456"]  // ← full ancestor chain
```

The result is a **DAG** (directed acyclic graph) — in practice almost always a tree.

### What this structure enables

**1. Correct thread reconstruction**

Even if mails arrive out of order (Bob's reply arrives before Alice's original because Carol's mailbox server was faster), the client can correctly reassemble the tree. The `references` list provides full ancestor information.

**2. Fork detection**

If two people reply in parallel to the same mail, a fork is created:

```
        Alice's mail
       /            \
   Bob's reply    Carol's reply  ← fork
       |
   Dave's reply
```

The UI can show this **as a tree** instead of linearly. Gmail collapses everything into a list; LibertyChat shows the real structure (with a toggle).

**3. Cryptographic continuity**

Every `message_id` is a hash over (content + sender identity pubkey + timestamp + parent IDs). A recipient can verify:

- "This mail claims to be a reply to `msg_abc123`."
- "It is signed by Bob's device."
- "Bob's device received `msg_abc123` and was able to reference it as a parent."

An attacker cannot inject forged replies into a thread (they do not know the content of the parent mails, cannot compute the hash).

### Thread-ID persistence through re-encryption

MLS rotates epoch secrets, but the `thread_id` is an **application-layer** ID that stays stable across epoch transitions. Even after 50 member add/remove operations and corresponding epoch transitions, the thread remains coherent for the user.

### Subject mutation in threads

Classical mail problem: the subject changes over the course of a thread ("Re:" prefixes, someone changes the subject mid-thread, Outlook appends subject suffixes). Threading via subject string is therefore fragile.

LibertyChat ignores the subject for threading entirely. The subject is **purely display-relevant**. Threading is based only on `thread_id` and `in_reply_to`/`references`. A user can change the subject mid-thread; the thread stays coherent.

UI convention: the **first subject** is displayed as the thread title; later subject changes are marked inline ("subject changed: …"). Optionally a user can set a "personal title" override per thread, valid only locally (CRDT-synced across their own devices, see Feature 16).

### Cross-thread searchability

Threads are searchable because:

- Subject strings are indexed client-side (or server-side via SSE, see Feature 18).
- Body full-text analogously.
- Thread boundaries are visible in search results — hits can be displayed as "thread with 8 mails, match in mail 3".

### Multi-device consistency

When Alice reads the same mail on phone and laptop, the read marker should be synchronous on both. The Read/Unread/Star/Archive flag is:

- **Not written into the mail itself** (would violate MLS Forward Secrecy, since the server cannot cleanly map a mutation onto an encrypted blob).
- **Instead held in a separate CRDT state** (see Feature 16) that syncs between Alice's own devices.
- Bob's devices do **not** see Alice's read status (privacy: Bob should not know whether Alice has read the mail — the user can separately opt into read receipts).

### Why this is superior

- **Gmail**: server-side threading based on subject + sender clustering. Works often, fails on mid-thread subject changes. Server knows the threading structure in plaintext.
- **Outlook**: conversation view based on `In-Reply-To` + subject. Rarely displays forks correctly. Plaintext threads.
- **mutt / Thunderbird**: correct tree threading, but unencrypted headers.
- **ProtonMail**: conversation view with subject matching; server does not see subject plaintext, but routing headers yes.
- **Matrix reply threading**: technically a DAG, but UI typically flat. No cryptographic hash verification of the parent relationship.
- **LibertyChat**: tree threading with **cryptographically authenticated parent relationships**, subject-independent, with multi-device CRDT sync for flags. The server sees neither subject nor threading structure — both are inside the encrypted application payload.

---

Walkthrough continues with further steps as the design discussion progresses.
