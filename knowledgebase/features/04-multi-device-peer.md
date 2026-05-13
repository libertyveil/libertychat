# Feature 4: Multi-Device — Every Device Is a First-Class Member

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Every device is its own MLS member with its own keypair, signed by the user's identity key. The sender uploads each message once; the mailbox server fans the encrypted blob out to all subscribed devices in parallel. No primary-secondary asymmetry; no master device required to be online.

## Why this is superior (preview)

- **SimpleX**: master-slave model — secondary devices are "remote views" of the primary.
- **Signal**: linked devices share derived keys with the primary; primary must approve.
- **WhatsApp**: late multi-device with limited group support.
- **Matrix**: every device has own keys but cross-signing and verification UX is fragile.
- **LibertyChat**: peer-equal devices, single upload, server-side fanout, log-N device add/remove.
