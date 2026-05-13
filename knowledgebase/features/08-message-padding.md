# Feature 8: Message Padding with Fixed Buckets

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

All messages padded to fixed sizes: **1 KiB, 4 KiB, 16 KiB, 64 KiB, 256 KiB**. Traffic analysis via message size becomes impossible — a "yes" looks the same as a longer reply. Optional cover traffic on top: regular dummy messages so that the mailbox server sees a constant data flow and cannot infer activity patterns.

## Why this is superior (preview)

- **Signal / WhatsApp**: variable-size encrypted payloads; sizes leak roughly how long a message is.
- **Matrix**: same — JSON-event size correlates with content length.
- **SimpleX**: limited padding, no fixed-bucket scheme.
- **LibertyChat**: fixed buckets eliminate the size channel entirely; cover traffic eliminates the timing channel.
