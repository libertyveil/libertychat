# Feature 14: Self-Destructing Messages with Cryptographic Guarantee

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Unlike "delete for everyone" which relies on client cooperation:

- Sender sets a TTL when sending.
- The recipient device stores the message in a hardware keychain with a trigger-delete policy.
- On TEE-capable hardware (Secure Enclave, StrongBox), the deletion is attested by the TEE.
- The sender receives a **cryptographic receipt** signed by the TEE confirming the delete was executed.

This gives "true" disappearance instead of "the client said it was deleted, we have no way to verify".

## Why this is superior (preview)

- **Signal / WhatsApp / Telegram**: disappearing messages enforced only by the client. A modified client can keep them forever.
- **LibertyChat**: hardware-attested deletion with a verifiable receipt. The sender knows the delete actually happened (on supporting platforms).
