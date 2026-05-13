# Feature 12: Social Recovery via Shamir Secret Sharing

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

The master identity key is split via Shamir Secret Sharing into N shares with a threshold M (e.g. 3-of-5). Shares are distributed to trusted contacts. If the user loses the master device, M contacts each send their share; the user reconstructs the key locally. No cloud backup, no service provider involvement, no centralized key escrow.

## Why this is superior (preview)

- **Signal**: PIN-based recovery via Signal-managed Secure Value Recovery infrastructure — Signal infrastructure required.
- **WhatsApp**: backup to iCloud / Google Drive, plaintext or platform-encrypted.
- **Matrix**: Secure Secret Storage in the homeserver — homeserver must be trusted or recovery passphrase memorized.
- **LibertyChat**: recovery requires only the cooperation of trusted humans, never a service provider.
