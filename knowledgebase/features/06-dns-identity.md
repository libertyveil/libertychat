# Feature 6: DNS-Based Identity Discovery

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Optional `@user@domain` discovery via signed DNS TXT records on a domain the user controls. No central name server, no `ns.jami.net` equivalent, no `@user:matrix.org` lock-in. If the user has no domain, identity stays invitation-link-only (SimpleX-style). If the user has a domain, federation-style discoverability without provider binding.

```
_libertyveil.example.com  TXT  "v=1;ipk=...;mailbox=lvm://..."
```

## Why this is superior (preview)

- **Matrix**: identity coupled to homeserver, moving servers means losing the handle.
- **Jami**: OpenDHT + central name server (`ns.jami.net`) for username registration.
- **SimpleX**: no discovery at all — privacy-preserving but inconvenient.
- **Signal**: phone number as identity, owned by the carrier and the platform.
- **LibertyChat**: identity owned by the user, discoverable through infrastructure they already control.
