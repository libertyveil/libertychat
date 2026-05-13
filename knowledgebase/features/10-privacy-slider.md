# Feature 10: Per-Conversation Privacy Slider

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

The user picks the privacy level per conversation:

| Level | Routing | Cover traffic | Latency |
|---|---|---|---|
| Casual | Direct (P2P preferred) | no | minimal |
| Private | Mailbox + TLS | no | low |
| High | Tor (3 hops) | no | medium |
| Maximum | Nym mix-net | yes | high |

Family chat at minimum latency, activist chat at maximum anonymity — both in the same app, no separate clients.

## Why this is superior (preview)

- **Signal / WhatsApp / Matrix / SimpleX**: single privacy posture per app, applied uniformly to all chats.
- **Session**: onion routing always-on, no opt-out for casual conversations.
- **LibertyChat**: user picks the trade-off per conversation, matching the threat model to the actual content.
