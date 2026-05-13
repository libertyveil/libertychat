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

Walkthrough continues with further steps as the design discussion progresses.
