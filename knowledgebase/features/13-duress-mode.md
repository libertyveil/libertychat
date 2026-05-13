# Feature 13: Plausible Deniability / Duress Mode

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Two passphrases for app decryption:

- **Real passphrase** — unlocks the actual profile with real contacts and messages.
- **Duress passphrase** — unlocks a decoy profile with innocuous contacts and messages.

When coerced to unlock (border crossing, search by adversary, physical threat), the user enters the duress passphrase. The adversary sees a plausible-looking but empty profile; the real data remains encrypted and unreachable without the real passphrase. The two profiles share no observable indicator of which is "real".

## Why this is superior (preview)

- **Signal / WhatsApp / Matrix / SimpleX**: a single unlock reveals everything.
- **VeraCrypt** has a similar hidden-volume concept for disks; LibertyChat applies it to messenger profiles.
- **LibertyChat**: structural defense against coerced unlocks, particularly relevant for journalists, activists, and travelers through hostile jurisdictions.
