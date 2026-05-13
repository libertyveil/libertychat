# Feature 17: Cuckoo-Filter-Based Push

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Instead of classical APNs/FCM-style "push notification addressed to device X" (which leaks per-event metadata to the push service):

- The push service holds a periodically-updated **Cuckoo-filter digest** representing "which users have new messages waiting".
- Devices poll the digest and check probabilistically whether they are affected.
- If yes, the device fetches from its mailbox queue.

The push service learns only that *somebody* has new messages, not *which user*. Massive reduction in metadata at the push layer, which is structurally the weakest link in iOS/Android messaging today (Apple/Google know who got a push notification, even when content is E2EE).

## Why this is superior (preview)

- **Signal / WhatsApp / Matrix / SimpleX**: classical per-user push tokens; the push provider can correlate user activity timing.
- **LibertyChat**: per-user push events are invisible to the push service.
