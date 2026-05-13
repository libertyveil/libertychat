# Feature 2: Hybrid Post-Quantum from Day One

Step-by-step walkthrough of how LibertyChat uses **hybrid classical + post-quantum cryptography** at every layer. Each step explains what happens and notes what makes this approach superior to the existing market.

LibertyChat pairs every classical primitive (X25519, Ed25519) with a post-quantum counterpart (ML-KEM-768, ML-DSA-65). A break in either family alone does not compromise the system.

---

## Step 1: Why classical cryptography is vulnerable

Today's encryption stack rests on two mathematical assumptions:

- **ECDH / X25519** — based on the **Discrete Logarithm Problem** over elliptic curves. Given `g` and `g^a`, find `a`. Exponentially hard on classical hardware.
- **EdDSA / Ed25519** — same underlying assumption: signatures are only secure because DLP is not efficiently solvable.

A sufficiently large quantum computer running **Shor's algorithm** breaks both in **polynomial time**. Once a "Cryptographically Relevant Quantum Computer" (CRQC) exists:

- Every ECDH key exchange becomes attackable — session keys can be reconstructed retroactively.
- Every Ed25519 signature becomes forgeable — identities and KeyPackages can be rebuilt.

NSA/NIST conservative timeline: **2030–2040** for a CRQC. Some estimates earlier. Nobody knows exactly, but every nation-state intelligence agency is planning for it.

### Why this matters

- **Signal / WhatsApp / Matrix / SimpleX**: all rely on ECDH-based handshakes at the foundation. The classical-only versions of these protocols are vulnerable to a future CRQC.
- **LibertyChat**: takes the threat seriously from the start, not as a later add-on.

---

## Step 2: Harvest Now, Decrypt Later

The real problem is not the CRQC arriving in 2035 — it is **today**.

The attack strategy openly documented by NSA/NIST and visible in surveillance disclosures:

1. **Today** — intelligence agencies collect encrypted internet traffic en masse (NSA Utah datacenter, GCHQ Tempora, etc.). TLS handshakes, messenger protocols — all recorded and stored.
2. **Today** — the traffic is unbreakable; classical crypto still holds.
3. **2035+** — CRQC becomes available. Archived traffic is retroactively decrypted.
4. **Result** — every message, every key exchange, every signature from today is readable in 10–15 years.

For messengers this is especially critical because:

- **MLS epoch secrets** are derived from key exchanges. If the KEX breaks retroactively, the entire Forward Secrecy chain collapses.
- **KeyPackages** are signed bundles that can sit in DNS or on mailbox servers for years. Forging an old signature lets an attacker rewrite history.
- **Long-form mail** has long retention. A mail from today must still be secret in 2040.

### Why this is superior

- **Signal**: added PQXDH for 1:1 in 2023 (good), but groups still rely on classical Sender Keys.
- **WhatsApp**: no PQ for messaging payloads yet.
- **Matrix**: PQ work-in-progress, not deployed.
- **SimpleX**: PQ planned, not yet shipped.
- **LibertyChat**: hybrid PQ across **every** cryptographic layer from v1.0. No layer left vulnerable.

---

## Step 3: NIST standardization — what exists today

NIST opened a public PQC competition in 2016. After 8 years and four rounds of cryptanalysis, the first standards were ratified in August 2024:

| Standard | Common name | Purpose | Basis |
|---|---|---|---|
| **FIPS 203** | **ML-KEM** | Key Encapsulation | Module-Lattice (Kyber) |
| **FIPS 204** | **ML-DSA** | Signatures | Module-Lattice (Dilithium) |
| **FIPS 205** | **SLH-DSA** | Signatures (hash-based) | Hash trees (SPHINCS+) |
| **FIPS 206** (in progress) | **FN-DSA** | Signatures | NTRU-Lattice (Falcon) |

For LibertyChat:

- **ML-KEM-768** for key exchange (formerly Kyber-768) — security level ~AES-192
- **ML-DSA-65** for signatures (formerly Dilithium-3) — security level ~AES-192

Properties of these lattice-based algorithms:

- Security based on **Learning With Errors (LWE)** and **Module-LWE** — no efficient quantum algorithm known.
- Performance: ML-KEM is **faster** than X25519 on encapsulation, comparable on decapsulation.
- Size trade-off:
  - Public key: ~1.2 KB (vs. 32 bytes for X25519) — **38× larger**
  - Ciphertext: ~1.1 KB (vs. 32 bytes) — **34× larger**
  - Signature (ML-DSA): ~3.3 KB (vs. 64 bytes for Ed25519) — **52× larger**

Acceptable cost for messenger payloads (already in the KB range).

### Why this is superior

- Other messengers either delay PQ adoption or use pre-standardization variants. LibertyChat uses the **finalized NIST standards** (FIPS 203/204) from v1.0 — no need for re-keying when standards change because there are no further changes.

---

## Step 4: What "hybrid" actually means

Nobody trusts the new PQ algorithms blindly. They have been standardized only since 2024; cryptanalysis is young compared to ECDH (40+ years of review). NIST-finalist algorithms have been broken before — **SIKE** reached round 4 of the NIST competition and was then broken in hours on a single laptop.

**Hybrid crypto = running two independent algorithms in parallel so that both must break for the system to fall.**

### Hybrid KEM (key exchange)

```
Sender:
  (ct_classical, ss_classical) = X25519.Encaps(pk_classical)
  (ct_pq,        ss_pq)        = ML-KEM.Encaps(pk_pq)
  ss_final = HKDF(ss_classical || ss_pq || transcript)
  send: ct_classical || ct_pq

Receiver:
  ss_classical = X25519.Decaps(sk_classical, ct_classical)
  ss_pq        = ML-KEM.Decaps(sk_pq, ct_pq)
  ss_final = HKDF(ss_classical || ss_pq || transcript)
```

`ss_final` is secure as long as **either** shared secret remains unknown. An attacker must break **both** algorithms.

### Hybrid signatures

```
Signer:
  sig_classical = Ed25519.Sign(sk_classical, msg)
  sig_pq        = ML-DSA.Sign(sk_pq, msg)
  signature = sig_classical || sig_pq

Verifier:
  ok_classical = Ed25519.Verify(pk_classical, msg, sig_classical)
  ok_pq        = ML-DSA.Verify(pk_pq, msg, sig_pq)
  valid = ok_classical AND ok_pq
```

Signature is valid only if **both** verifications pass.

### Why this is superior

- **Signal PQXDH**: hybrid X25519 + Kyber for 1:1 initial exchange only. Group key exchange in Sender Keys is still classical.
- **Apple iMessage PQ3**: hybrid only at initial key establishment, not at every layer.
- **TLS 1.3** `X25519Kyber768Draft00`: hybrid handshake only; no signature hybrid.
- **LibertyChat**: hybrid at **every layer** — KEX, signatures, identity, MLS internal HPKE — uniformly applied.

---

## Step 5: Where hybrid PQ is used in LibertyChat

Hybrid PQ is applied at **every** cryptographic touchpoint. A single classical-only operation would be enough to compromise the chain.

### 5.1 MLS ciphersuite

MLS (RFC 9420) defines **ciphersuites** as interchangeable crypto profiles. LibertyChat uses the standardized hybrid suite:

```
MLS_256_XWING_AES256GCM_SHA512_Ed25519+MLDSA65
```

Components:

- **HPKE-KEM**: X-Wing (the official hybrid combiner for X25519 + ML-KEM-768)
- **Symmetric AEAD**: AES-256-GCM (or ChaCha20-Poly1305 alternative)
- **Hash**: SHA-512
- **Signatures**: Ed25519 + ML-DSA-65

Every TreeKEM path encryption, every Welcome message, every Commit signature runs through this hybrid stack.

### 5.2 KeyPackages

Each KeyPackage contains:

- Classical HPKE pubkey (X25519)
- PQ HPKE pubkey (ML-KEM-768)
- Classical identity pubkey (Ed25519)
- PQ identity pubkey (ML-DSA-65)
- Double signature over the entire bundle (Ed25519 + ML-DSA)

Per-KeyPackage size: ~5–6 KB instead of ~200 bytes classical-only. Acceptable.

### 5.3 Identity keys (long-term)

Each user holds **four** identity keys on each device:

- Ed25519 sign key (classical)
- ML-DSA-65 sign key (PQ)
- X25519 encrypt key (classical, for offline encryption without KEM)
- ML-KEM-768 encrypt key (PQ)

Generated at account creation; root of trust for the entire identity.

### 5.4 Mailbox queue auth

Here **classical only** (Ed25519). Rationale:

- Queue auth keys are short-lived (per-queue, rotate via VRF).
- No long-term confidentiality needed — even if signatures become forgeable in 20 years, the queue is long retired.
- Bandwidth overhead on every read/write would not be worth it.

Conscious decision: PQ where long-term confidentiality or long-term identity matters; classical where only short-term auth matters.

### 5.5 Realtime media (WebRTC / DTLS-SRTP)

DTLS handshake becomes hybrid once TLS-WG finalizes the hybrid cipher suites (in `draft-ietf-tls-hybrid-design`). Until then, **classical handshake + PQ-tunnel-wrapping** of the DTLS connection over a pre-established MLS key (defense in depth).

### 5.6 File chunks (XFTP)

File master key is exchanged through the MLS channel → inherits hybrid PQ automatically. Chunks themselves use AEAD (ChaCha20-Poly1305) — symmetric crypto only loses **effective half** of key length to Grover's algorithm. Solved by using 256-bit keys (effective 128-bit against quantum, still secure).

### Why this is superior

- **No layer left untouched**: every other deployed messenger has at least one critical layer (KEX, signatures, or media) still classical-only.
- **Defense in depth**: even at layers where the protocol does not yet support PQ natively (DTLS-SRTP), LibertyChat wraps PQ around it.
- **Conscious trade-offs**: PQ is not applied where it would only add overhead without long-term benefit (short-lived queue auth). Decisions are explicit, not accidental.

---

Walkthrough continues with further steps as the design discussion progresses.
