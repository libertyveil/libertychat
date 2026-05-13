# Feature 19: Anti-Spam via Anonymous Rate-Limit Tokens

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Spam protection typically requires identity (account-based rate limits). LibertyChat instead uses **Privacy Pass-style anonymous tokens**:

- An issuer (e.g. the recipient's mailbox server) hands out blind-signed tokens to authenticated senders or to anyone who solves a proof-of-work / human-verification challenge.
- The sender attaches a token to each write attempt.
- The mailbox server validates the token signature without learning which sender presented which token.
- If abuse occurs, the issuer can revoke entire token batches without identifying anyone.

Spam protection without identity tracking.

## Why this is superior (preview)

- **Signal / WhatsApp**: rate-limits tied to phone-number identity.
- **Matrix**: rate-limits per Matrix ID, which is tied to a homeserver.
- **SimpleX**: limited spam protection model.
- **LibertyChat**: enforces sender good-behavior cryptographically without ever learning who the sender is.
