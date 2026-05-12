# Technology stack

## Core language: Rust

Rationale:

- **Memory safety** without garbage collection: deterministic resource use important for both server-side high-load and mobile battery constraints
- **Strong type system**: catches protocol-state-machine bugs at compile time
- **Cryptographic ecosystem**: RustCrypto, ring, libsodium-rs, mls-rs are production-grade
- **Mobile FFI**: cleaner integration with Swift (UniFFI) and Kotlin (JNI) than Haskell
- **Compile target diversity**: native binaries for x86_64, ARM, also WASM
- **Hiring pool**: larger than Haskell, easier to attract contributors
- **Single language across stack**: server, client core, CLI all in Rust; mobile UI thin wrappers

Trade-off acknowledged:
- Rust has a learning curve. Onboarding contributors takes longer than Go or Python.
- Compile times can be slow (~5-10 minutes for full build). Mitigated by `sccache` and incremental compilation.

## Crate dependencies (planned)

### Cryptography

| Crate | Purpose |
|---|---|
| `mls-rs` (AWS) | MLS RFC 9420 implementation |
| `x25519-dalek` | X25519 ECDH |
| `ed25519-dalek` | Ed25519 signatures |
| `pqcrypto-kyber` | CRYSTALS-Kyber post-quantum KEM |
| `pqcrypto-dilithium` | CRYSTALS-Dilithium post-quantum signatures |
| `chacha20poly1305` | AEAD cipher |
| `blake3` | Fast hash |
| `hkdf` | Key derivation |
| `argon2` | Password-based KDF |

### Networking

| Crate | Purpose |
|---|---|
| `quinn` | QUIC implementation |
| `rustls` | TLS 1.3 (used by quinn) |
| `tokio` | Async runtime |
| `webrtc-rs` or `str0m` | WebRTC for media |
| `arti-client` | Built-in Tor client (no system tor needed) |

### Storage

| Crate | Purpose |
|---|---|
| `sled` or `redb` | Embedded key-value store for server |
| `rusqlite` | SQLite for client-side message store |
| `serde` + `bincode` | Serialization |

### CLI / UI

| Crate | Purpose |
|---|---|
| `clap` | CLI argument parsing |
| `ratatui` | TUI for CLI client |
| `tauri` or `iced` | Desktop GUI (decision pending) |

### FFI for mobile

| Tool | Purpose |
|---|---|
| `uniffi` | Mozilla's tool for generating Swift/Kotlin bindings from Rust |
| `cargo-ndk` | Build Android-targeted Rust binaries |

## Repository structure

Workspace with multiple crates:

```
libertychat-core/        ← protocol library (shared by server and clients)
  Cargo.toml
  src/
    identity.rs           ← identity keypairs, descriptors
    crypto.rs             ← primitive wrappers
    ratchet.rs            ← Double Ratchet
    mls.rs                ← MLS group bindings
    mailbox.rs            ← mailbox protocol client
    mail.rs               ← persistent mail format
    realtime.rs           ← WebRTC signaling helpers

libertychat-server/      ← mailbox server daemon
  src/
    main.rs
    transport.rs          ← QUIC + TLS listeners
    queue.rs              ← queue storage + auth
    storage.rs            ← persistent backend (sled)

libertychat-relay/       ← TURN relay (optional separate deployment)
  src/
    main.rs

libertychat-cli/         ← terminal client
  src/
    main.rs

libertychat-desktop/     ← GUI client (later)
  src/
    main.rs

ffi/                     ← FFI bindings for mobile
  swift/                  ← generated Swift bindings
  kotlin/                 ← generated Kotlin bindings
```

## Build and CI

- `cargo` for builds
- `nix flake` for reproducible toolchain
- GitHub Actions for CI (test, build, lint, security audit)
- `cargo audit` for known vulnerabilities in dependencies
- `cargo deny` for license compliance
- `rustsec` advisories integration

## Mobile builds

- **iOS**: build core as static library; consume from Swift via UniFFI. Wrap in SwiftUI or AppKit shell.
- **Android**: build core via `cargo-ndk` for arm64-v8a / armeabi-v7a / x86_64; consume from Kotlin via UniFFI/JNI. Wrap in Jetpack Compose shell.

Both mobile UIs are thin: business logic in Rust core, UI just renders state.

## Desktop GUI

Two candidates:

### Tauri
- Rust backend, web frontend
- Familiar to web devs
- Smaller binary than Electron (~10 MB vs 100+ MB)
- Cross-platform: Linux, macOS, Windows
- Native webview (uses system webview, not bundled Chromium)

### Iced
- Pure Rust, no web stack
- Native rendering (Skia or wgpu)
- Smaller dependency tree
- Newer, less mature than Tauri

Decision pending. Tauri is safer choice for v1.0 (faster to build a polished UI). Iced for later if we want to drop the web layer.

## Server deployment

- Single binary ~20 MB
- Optional Docker image for easy deployment
- Systemd unit file shipped
- Configuration via TOML file or env vars
- Logs to stdout (12-factor app style)
- Optional Prometheus metrics endpoint

## Testing

- Unit tests for crypto primitives (test vectors from upstream specs)
- Integration tests with simulated network
- Fuzzing of wire format parsers (`cargo-fuzz`)
- Property-based testing with `proptest`
- Compatibility tests against reference implementations (when feasible)

## Documentation

- `cargo doc` for API docs
- `mdbook` for user-facing documentation
- Spec documents in this repo, kept in sync with implementation
- Tutorial walkthroughs (writing a client, running a server)

## Licensing

All code under **AGPL-3.0-or-later**. Key reasons:

- Network use clause prevents SaaS-isation without source contribution
- Compatible with using AGPL libraries (some PQ crypto, mls-rs is Apache, generally compatible)
- Strong copyleft aligns with project values

Trademark policy: "LibertyChat" name and logo are not in the AGPL license. Forks must rename. Prevents bad actors from running compromised forks under the original brand.
