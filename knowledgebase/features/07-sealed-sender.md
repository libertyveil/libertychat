# Feature 7: Sealed Sender

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Sender identity is encrypted under the recipient's public key inside the outer envelope. The mailbox server sees that "someone wrote into queue X" but cannot determine who. Combined with multi-server distribution, the server cannot map a network of sender-recipient relationships.

## Why this is superior (preview)

- **Signal**: pioneered Sealed Sender (2018); we adopt the concept.
- **WhatsApp / Matrix / SimpleX**: server can typically see at least the sender's identifier or queue-write authentication.
- **LibertyChat**: sealed sender by default for every write; combined with VRF queue rotation, social-graph mapping is structurally prevented.
