# Feature 3: Hybrid Delivery — P2P-First + Mailbox-Fallback

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Direct P2P delivery when both parties are online and NAT-traversable; mailbox-queue fallback otherwise. The user notices no difference. Metadata exposure to the mailbox server is minimized whenever P2P succeeds — at best, zero metadata reaches any server.

## Why this is superior (preview)

- **Signal / WhatsApp**: every message routes through central servers, even when both parties are next to each other.
- **Matrix**: server-to-server federation always involved.
- **SimpleX**: mailbox-only, never direct.
- **Jami**: P2P-only, struggles when peers are not simultaneously online.
- **LibertyChat**: combines the best of both — direct when possible, queued when needed.
