# Feature 18: Encrypted Searchable Index (Server-Side SSE)

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Searchable Symmetric Encryption (SSE) lets the mailbox server perform full-text search over an encrypted index without ever seeing plaintext:

1. Client builds a per-user encrypted index of mail content.
2. To search, client generates a query token from the search term and a per-user key.
3. Server uses the token to look up matching encrypted entries.
4. Server returns encrypted hits; client decrypts the final results.

The server learns "the client searched for *something*" but never the search term, never the content, never the match positions in the clear.

## Why this is superior (preview)

- **Signal / WhatsApp**: search is purely client-side, requiring full message history to be on the searching device.
- **Matrix**: server-side search exists but the server sees plaintext (since it stores the message content for unencrypted rooms; encrypted rooms have no server search).
- **LibertyChat**: full-text search over years of mail history at server speed, without the server ever decrypting anything.
