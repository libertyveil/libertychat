# Persistent mail layer

## Why a mail layer?

Most messengers conflate two distinct types of communication:

- **Ephemeral chat**: short bursts, often in real time, low information density per message
- **Persistent correspondence**: longer messages, formal, expected to be retained and re-read

Email handles the second well but is a privacy disaster (plaintext, metadata leaks, spam, phishing).

LibertyChat treats both as first-class. Same encrypted infrastructure, different presentation.

## Concept

A **mail message** is a long-form, persistent message stored encrypted on the user's mailbox server. Properties:

- Subject line (encrypted, like the body)
- Threaded (replies form a tree)
- Attachments (any size, via the file storage layer)
- Searchable client-side (server stores ciphertext blobs only)
- Long retention by default (years, not days)
- Available across all the user's devices

## Format

Internally, a mail message is a structured payload:

```
{
  "version": 1,
  "from": "<sender's identity public key>",
  "to": ["<recipient identity public key>", ...],
  "subject": "<utf-8 string>",
  "body": "<markdown text>",
  "attachments": [
    {
      "name": "<filename>",
      "size": 12345,
      "mime_type": "application/pdf",
      "file_chunk_refs": [...]   # references into XFTP-style file storage
    }
  ],
  "in_reply_to": "<message id>",  # optional, threading
  "references": ["<message id>", ...],  # full thread chain
  "created_at": "<ISO 8601 timestamp>",
  "signature": "<sender signature over the above>"
}
```

The whole structure is encrypted with the sender↔recipient ratchet key before being transmitted.

## Storage

Mail messages are stored on the recipient's mailbox server, separate from the chat queues:

- Persistent retention (configurable, default unlimited)
- Encrypted at rest (the server cannot decrypt; only the user's devices have the key)
- Client periodically syncs new mail and caches it locally

This is analogous to IMAP's server-side mail store, but with E2EE and no plaintext metadata.

## Threading and conversations

Replies link to their parent via `in_reply_to`. Clients reconstruct conversation trees.

A thread is identified by the root message ID. Subject changes within a thread are tracked but don't fork the thread.

## Search

The user's local devices maintain a **decrypted local index** for full-text search. Server has no search capability (no plaintext available). When a new device joins, it syncs ciphertext from the server and rebuilds the local index.

Storage: ~10× the size of the original mail volume for index data. Acceptable on modern devices.

## Folders / labels

Client-side: users organize mail with **labels** (Gmail-style). Labels are stored as encrypted metadata in the same envelope.

Server has no concept of folders.

## Signatures and authenticity

Every mail message is signed by the sender's identity key (Ed25519+Dilithium hybrid). Recipient verifies before displaying.

A signed message is non-repudiable: the sender cannot later deny having sent it. This is a feature (legal documents, agreements) and a bug (whistleblowers cannot deny). For deniable mail, an alternative mode uses MAC-based authentication (deniable signature) — selectable per message.

## Spam prevention

LibertyChat mail does not have an open delivery mechanism. To send mail to someone, you must first have an established connection (via invite link or existing contact).

Connection-by-discovery (`@user@domain` lookup via DNS) requires the recipient to accept the connection request first. Until then, no mail can be delivered.

This eliminates spam by construction, at the cost of requiring an out-of-band first contact.

## Differences from email

| | Email (SMTP/IMAP) | LibertyChat Mail |
|---|---|---|
| Encryption | optional (PGP/S/MIME) | mandatory E2EE |
| Metadata visibility | server reads sender/recipient/subject | server sees nothing meaningful |
| Spam | massive problem | structurally impossible |
| Identity | name@domain (via SMTP) | cryptographic public key |
| Federation | universal (any SMTP server) | universal (any LibertyChat mailbox) |
| Threading | RFC 5322 References header | first-class thread IDs |
| Long retention | yes (server-side) | yes (server-side, encrypted) |
| Open standard | RFC | open standard (will be) |

## Migration from email

Long-term goal: a bridge that imports email accounts into LibertyChat mail. Inbound email decrypted by user's keys before storage; outbound email signed and sent through standard SMTP. This lets users wean off plaintext email gradually.

Out of scope for v1.0; planned for v2.x.

## Why this matters for the project

A pure messenger is a niche tool. Email-style persistent correspondence is universal but legacy. By unifying both in one E2EE stack, LibertyChat aims to become the user's complete communication tool, replacing both Signal-style chat and Gmail-style mail.
