# Feature 16: CRDT-Based Multi-Device Sync for Local State

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Local state that needs to sync between a user's devices — read markers, drafts, conversation ordering, labels — is replicated as Conflict-free Replicated Data Types (Automerge or Yjs). Each device modifies the CRDT locally; changes merge automatically on reconnect without conflict, regardless of the order in which devices come online.

Encrypted under the user's device-group MLS keys before transit, so the mailbox server never sees the state.

## Why this is superior (preview)

- **Signal**: minimal cross-device state sync — read markers exist but drafts and labels do not sync.
- **WhatsApp**: cross-device sync via primary device; offline edits awkward.
- **Matrix**: account-data on homeserver; conflict resolution is last-writer-wins.
- **LibertyChat**: rich local state syncs reliably offline-first, with no central reconciliation point.
