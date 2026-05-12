# Roadmap

This document outlines the phases of work needed to take LibertyChat from concept to production.

Each phase has a goal, a rough scope, and an estimated effort range. Estimates assume a small dedicated team (3-5 contributors).

## Phase 0: Concept and specification (current)

**Goal**: Lock down the design enough that implementation can proceed.

**Deliverables**:
- This repo (concept + design documents) — done
- Public RFC process for protocol changes
- Initial set of "design decision" documents
- Threat model written down formally (TLA+ sketches optional)

**Effort**: 2-3 months
**Status**: in progress

## Phase 1: Core library MVP (`libertychat-core`)

**Goal**: A Rust library that implements the protocol's lower layers, sufficient for a simple CLI client.

**Scope**:
- Identity management (key generation, certificates)
- Mailbox protocol client (CREATE / READ / WRITE / ACK)
- Double Ratchet (one-to-one sessions)
- Initial PQ hybrid handshake
- Basic message format (text only, no MLS yet)
- Local storage (SQLite)

**Out of scope for this phase**:
- Groups (MLS)
- Realtime A/V
- Mail layer
- Mobile bindings

**Deliverables**:
- `libertychat-core` crate publishable on crates.io
- Test vectors and fuzz suite
- API documentation

**Effort**: 4-6 months

## Phase 2: Server (`libertychat-server`)

**Goal**: A self-hostable mailbox server.

**Scope**:
- QUIC + TLS 1.3 listeners
- Queue storage (sled-backed)
- Authentication via per-queue keys
- Subscription / push notification support
- Configuration system (TOML)
- Systemd integration
- Docker image

**Deliverables**:
- `libertychat-server` binary
- Deployment guide
- Reference public servers (operated by community)

**Effort**: 3-4 months
**Can run partially in parallel with Phase 1.**

## Phase 3: CLI client (`libertychat-cli`)

**Goal**: A complete terminal-based client.

**Scope**:
- Connect to one or more mailbox servers
- Send/receive text messages
- Manage contacts, identities, devices
- Create and accept invitations
- TUI built on `ratatui` for non-power users
- Scriptable mode for automation

**Deliverables**:
- `libertychat-cli` binary
- Tutorial walkthrough
- Comparison test against `libertychat-core` API

**Effort**: 2-3 months

## Phase 4: Groups via MLS

**Goal**: Add group chat support using MLS.

**Scope**:
- Integrate `mls-rs` into `libertychat-core`
- Group lifecycle: create, invite, remove, leave, dissolve
- Roles: owner, admin, member
- Group invitation links
- Message routing through group queues

**Deliverables**:
- MLS group support in `libertychat-core` and clients
- Group test scenarios

**Effort**: 2-3 months

## Phase 5: Realtime A/V

**Goal**: Audio and video calls.

**Scope**:
- Integrate `webrtc-rs` (or `str0m`) into `libertychat-core`
- Signaling via existing chat channels
- TURN server implementation (`libertychat-relay`)
- 1-on-1 calls
- Group calls (full mesh up to 6, SFU for larger)

**Deliverables**:
- Working A/V in CLI client (audio only) and Desktop client (audio + video)
- Reference TURN relay
- Performance benchmarks

**Effort**: 4-6 months

## Phase 6: Mail layer

**Goal**: Persistent long-form mail.

**Scope**:
- Mail message format (subject, body, attachments, threading)
- Long-term server-side storage
- Client-side full-text search index
- Mail-specific UI in clients

**Deliverables**:
- Mail send/receive working
- Threading view
- Attachment handling

**Effort**: 2-3 months

## Phase 7: Desktop GUI

**Goal**: A polished desktop client.

**Scope**:
- Tauri-based GUI shell (or Iced — decided in Phase 5)
- Chat, mail, calls integrated
- Cross-platform: Linux, macOS, Windows
- Auto-update mechanism
- Notifications

**Deliverables**:
- `libertychat-desktop` for all three OSs
- Installation packages (deb, dmg, msi)
- AppImage / Flatpak for Linux

**Effort**: 4-6 months

## Phase 8: Mobile clients

**Goal**: iOS and Android apps.

**Scope**:
- UniFFI bindings to `libertychat-core`
- Native UI: SwiftUI for iOS, Jetpack Compose for Android
- Push notification integration
- Background sync
- Battery optimization

**Deliverables**:
- App Store and Google Play submissions
- F-Droid build for Android

**Effort**: 6-9 months
**Most effort concentrated here; mobile is the hardest platform.**

## Phase 9: Onion routing integration

**Goal**: Optional Tor and Nym transport.

**Scope**:
- Integrate `arti-client` for Tor
- Integrate `nym-sdk` for Nym (when stable)
- Per-conversation privacy levels
- Onion-only mode

**Deliverables**:
- Tor mode toggle in all clients
- Nym mode toggle in clients (when available)

**Effort**: 2-3 months
**Can run after Phase 5.**

## Phase 10: Federation refinement

**Goal**: Robust federation between servers.

**Scope**:
- Cross-server group operations
- Forwarded mail between domains
- Server-to-server discovery protocols
- Anti-abuse mechanisms

**Effort**: 3-4 months

## Phase 11: Audit and hardening

**Goal**: External audit and pre-1.0 polish.

**Scope**:
- Crypto audit by Trail of Bits / Cure53 / equivalent
- Penetration testing of server
- Bug bounty program launch
- Performance optimization
- Documentation pass

**Deliverables**:
- Audit reports public
- Fixes applied
- Stable 1.0 release

**Effort**: 4-6 months

## Total estimated effort (rough)

Sum of phases (with some overlap): **30-50 person-months**.

Concretely:
- 1 dedicated Rust developer: ~3 years
- Team of 3-5 dedicated contributors: ~12-18 months
- Plus design/specs/audit overhead: add 6 months either way

This is comparable to other privacy messenger projects: Signal took years, Matrix took years, SimpleX took years. There is no shortcut.

## Funding model

To attract contributors and reach the audit phase, the project needs funding. Options:

- Donations (OpenCollective, GitHub Sponsors)
- Grant applications (NLnet Foundation, OTF, Sovereign Tech Fund)
- Volunteer-led with no funding (slowest, most fragile)

No cryptocurrency-token-based funding model. No commercial licensing. No paid tiers. AGPL only.

## What's deliberately not on the roadmap

- **Cryptocurrency integration** — out of scope by principle
- **AI features** — out of scope; messenger is for human communication
- **Walled-garden ecosystem** — protocol is open, anyone can build clients
- **Phone number identity** — never
- **Centralized identity registry** — never (only optional DNS-based discovery)
