# Feature 5: Persistent Mail Layer

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Long-form, threaded, persistent messages stored encrypted on the user's mailbox server. Subject lines, threading, attachments of unlimited size via XFTP. Same MLS encryption stack as chat. Client-side full-text search. Replaces both messenger and email in one application.

## Why this is superior (preview)

- **Signal / WhatsApp / SimpleX**: no persistent long-form layer; everything is ephemeral chat-style.
- **Matrix**: rooms are persistent but not threaded mail-style.
- **Delta Chat**: uses SMTP/IMAP as transport, inheriting email's privacy problems.
- **LibertyChat**: dedicated mail channel with messenger-grade encryption, no email infrastructure leakage.
