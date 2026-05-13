# Feature 9: Ephemeral Queue Rotation (VRF-Based)

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Queue IDs rotate on a schedule, derivable only by contacts via a Verifiable Random Function. The mailbox server cannot link rotated queue IDs back to the same conversation. Contacts compute the current queue ID locally from a shared secret.

```
Day 1: queue_id = VRF(shared_secret, "2026-05-12")
Day 2: queue_id = VRF(shared_secret, "2026-05-13")
...
```

The server sees a continuous stream of new ephemeral queues and cannot correlate them.

## Why this is superior (preview)

- **SimpleX**: queues are static — one queue per contact lasts for the lifetime of the contact relationship.
- **Signal / WhatsApp**: account-based delivery; no queue concept.
- **Matrix**: rooms have stable IDs visible to the homeserver indefinitely.
- **LibertyChat**: the queue identifier is a moving target only the participants can compute, breaking long-term correlation at the server.
