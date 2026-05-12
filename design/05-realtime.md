# Realtime: audio and video calls

## Goals

- Low-latency 1-on-1 audio and video calls
- Group calls up to 12 participants (later: more via SFU)
- E2EE for media streams
- Direct peer-to-peer when possible; relay fallback otherwise
- Self-hostable relay infrastructure

## Stack

WebRTC is used for media. Specifically:

- **Codecs**: Opus for audio, VP8/VP9/AV1 for video, H.264 for hardware-accelerated devices
- **Transport**: SRTP over DTLS (RFC 5764), encrypted with keys derived from the chat session
- **Signaling**: LibertyChat's mailbox layer carries SDP offers/answers and ICE candidates as encrypted application messages
- **NAT traversal**: ICE with STUN and TURN

Rust libraries:
- `webrtc-rs` for the WebRTC stack
- `str0m` as alternative (smaller, more recent, may be preferred)

## Call setup

1. Caller's app constructs SDP offer with ICE candidates.
2. Encrypts and sends through the existing chat session (Double Ratchet).
3. Recipient's app rings, user accepts.
4. Recipient sends SDP answer + their ICE candidates.
5. Both sides perform ICE connectivity checks; pick best path.
6. DTLS handshake establishes media keys.
7. SRTP carries audio/video.

This reuses the existing E2EE chat channel for signaling — no separate signaling server needed.

## E2E media encryption

Media streams are E2EE between caller and recipient:

- DTLS-SRTP keys derived during the WebRTC handshake
- Even if media goes through a TURN relay, the relay sees only encrypted SRTP packets
- Optional **SFrame** layer for group calls (E2E encryption of media frames at the application layer, even when an SFU relays them)

## Group calls

For groups up to ~6 participants: full mesh (each peer connects to each other).

For larger groups: an **SFU (Selective Forwarding Unit)** is used. Each participant uploads once to the SFU; SFU forwards to all others. Bandwidth scales linearly per uploader.

LibertyChat's reference SFU implementation will be:

- Rust-based (built on `webrtc-rs` or integrated with existing OSS like Galene)
- Supports SFrame for E2E group media encryption
- Self-hostable (single binary)

Compatible with any SFU that supports MatrixRTC's SFrame profile (so users can plug in LiveKit or Galene if preferred).

## TURN relay

For NAT traversal when direct P2P fails:

- LibertyChat ships its own TURN server implementation in Rust
- Standard TURN protocol (RFC 8656)
- Supports TURN-over-TLS and TURN-over-UDP
- Self-hostable

Public TURN server fleet operated by community contributors; users can self-host or use commercial TURN providers if needed.

## Call quality and adaptation

- Adaptive bitrate based on network conditions
- Simulcast for video (sender encodes multiple resolutions, receiver picks)
- Echo cancellation, noise suppression via `webrtc-rs` audio processing module
- Hardware-accelerated codecs where available (mobile chipsets, recent Intel/AMD CPUs)

## Asynchronous voice and video messages

In addition to live calls:

- **Voice messages**: recorded audio file (Opus, ~32 kbps), sent as attachment via mailbox
- **Video messages**: short recorded video clip (VP9, capped at 30 seconds default), sent as attachment

Both are E2EE like any other file; recipient receives whenever they next come online.

This is what mainstream messengers call "voice notes" / "video notes" and is critical for asynchronous use.

## Comparison with existing systems

| | LibertyChat | SimpleX | Element/Matrix | Signal |
|---|---|---|---|---|
| Live audio | direct + TURN fallback | direct + TURN fallback | LiveKit SFU | TURN-based |
| Live video | direct + TURN fallback | direct + TURN fallback | LiveKit SFU | TURN-based |
| Group calls | own SFU + LiveKit interop | limited (~12) | LiveKit SFU | growing support |
| E2E group media | SFrame | not yet | SFrame in progress | E2E for groups |
| Voice messages | yes | yes | yes | yes |
| Video messages | yes | not yet | not standard | yes (limited) |

## Privacy notes

- TURN relays see encrypted media bytes, IPs of both endpoints, call duration. Self-hosting eliminates external observation.
- Without TURN (direct P2P): each peer learns the other's IP. With TURN-only mode (forced relay): IPs of both peers visible only to TURN server.
- Tor cannot carry WebRTC media (latency too high). For maximum anonymity, use voice/video messages (asynchronous) instead of live calls.
