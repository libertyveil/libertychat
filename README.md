# LibertyChat

A self-sovereign communication protocol and application stack — text, audio, video, and persistent mail-style messaging without trusted middlemen.

## Status

Concept phase. This repository contains design documents and protocol specifications. No production code yet.

## Vision

Communication should work like cryptocurrency: cryptographically guaranteed, no third party that must be trusted, and free from any single operator's control. LibertyChat aims to provide a complete messaging stack — chat, calls, files, persistent inbox — built on this principle.

The reference implementation will be in **Rust**, drawing inspiration from SimpleX (asynchronous mailbox model, no global identity), Matrix (federated rooms, MLS for groups), and Signal (Double Ratchet, Post-Quantum hybrid crypto), while addressing the limitations of each.

## Documents

- [`VISION.md`](VISION.md) — Mission, principles, target users
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — High-level system architecture
- [`COMPARISON.md`](COMPARISON.md) — How LibertyChat differs from existing messengers
- [`ROADMAP.md`](ROADMAP.md) — Phased implementation plan
- [`design/`](design/) — Detailed design documents per subsystem

## How to contribute

This is currently a solo design effort. Open an issue or pull request to discuss any of the documents. The protocol design is open to substantial change while in concept phase.

## License

AGPL-3.0-or-later. The protocol specifications are intended to be implementable by anyone, but server software derived from this project must remain open under AGPL.
