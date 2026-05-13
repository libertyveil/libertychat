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

## Step 6: Performance and size consequences in practice

Hybrid PQ costs bandwidth, storage, and CPU in exchange for quantum resistance. Concrete numbers:

### Size comparison per operation

| Operation | Classical (X25519/Ed25519) | Hybrid (+ML-KEM/ML-DSA) | Factor |
|---|---|---|---|
| Public key | 32 B | 1,216 B | ~38× |
| KEM ciphertext | 32 B | 1,120 B | ~35× |
| Signature | 64 B | 3,373 B | ~52× |
| Identity bundle (4 keys + 2 sigs) | ~200 B | ~5,500 B | ~28× |
| Complete KeyPackage | ~250 B | ~6 KB | ~24× |
| MLS Welcome (10-member group) | ~2 KB | ~25 KB | ~12× |
| MLS Commit (10-member group) | ~1 KB | ~15 KB | ~15× |

### CPU comparison (order of magnitude)

On a current smartphone SoC (Apple A17 / Snapdragon 8 Gen 3):

| Operation | Classical | Hybrid | Factor |
|---|---|---|---|
| KEM encapsulation | ~50 µs | ~80 µs (X25519 + ML-KEM-768) | 1.6× |
| KEM decapsulation | ~50 µs | ~70 µs | 1.4× |
| Sign | ~30 µs | ~250 µs | ~8× |
| Verify | ~80 µs | ~150 µs | ~2× |

Generating a hybrid signature costs ~250 µs instead of 30 µs. At one message per second, unnoticeable. In a 1000-member group with a massive Commit burst (every member signs a Welcome confirmation), it becomes relevant — still in the millisecond range.

### Where it pinches

- **Mobile push notifications with payload**: APNs/FCM cap payloads at 4 KB. A hybrid Welcome no longer fits as a complete push payload. Solution: push carries only a "you have mail" signal; the device fetches the Welcome from its mailbox queue.
- **DNS TXT for KeyPackages**: DNS TXT records are limited to 255 characters per string and ~64 KB per record. A classical KeyPackage fits a single record; a hybrid KeyPackage needs multiple records or must be referenced via URL.
- **QR code size for invite links**: a QR code embedding a hybrid public key grows (version 20+ instead of version 10). Still scannable, but denser.
- **Battery**: negligible in normal use. No measurable difference in 24h background operation.

### Where it does not pinch

- General bandwidth: messenger traffic is minimal compared to images/video. A conversation with ~100 messages/day consumes ~600 KB hybrid overhead per day.
- Storage: a KeyPackage pool on the mailbox server for 1,000 users with 10 KeyPackages each takes ~60 MB instead of ~2.5 MB. Trivial.

### Why this is superior

- **Conscious trade-offs**: every size consequence is explicitly accounted for and absorbed at a layer where it does not hurt (e.g. push payload becomes a signal-only push).
- Messengers retrofitting PQ later have to make these trade-offs **after the fact**, often with compatibility workarounds. LibertyChat plans for hybrid sizes from the start.

---

## Step 7: Cipher agility — preparing for "what comes after ML-KEM and ML-DSA"

Today's hybrid is X25519 + ML-KEM-768. But:

- What if ML-KEM is broken in 5 years (the way SIKE suddenly fell)?
- What if NIST standardizes a stronger PQ algorithm in 2030 (e.g. a code-based one as a backup to lattice-based ones)?
- What if X25519 falls faster than expected?

Answer: **cipher agility is built into the protocol itself**, not bolted on later.

### What cipher agility concretely means

Every message, every KeyPackage, every MLS Commit carries an explicit **ciphersuite identifier** in its header:

```
ciphersuite_id: u16  // e.g. 0xF031 = MLS_256_XWING_AES256GCM_SHA512_Ed25519+MLDSA65
```

On receipt the client checks:

1. "Do I know this ciphersuite?" → yes, continue / no, reject with a clear error.
2. "Are the contained algorithms still classified as secure?" → local allow-list.
3. "Do I prefer a stronger suite?" → upgrade on re-keying.

### Three mechanisms for smooth migration

**1. Multi-cipher KeyPackages**

A device can publish **several KeyPackages in parallel**, one per supported ciphersuite. Example:

- KeyPackage A: `MLS_256_XWING_AES256GCM_SHA512_Ed25519+MLDSA65` (today's default)
- KeyPackage B: `MLS_256_FUTURE_CIPHER_...` (as soon as available)

The initiator picks the strongest suite both sides support.

**2. In-group ciphersuite upgrade**

MLS supports **Reinit Commits**: an existing group can atomically migrate to a new ciphersuite. All members swap their KeyPackages, the next Commit is in the new suite, old epoch secrets are either archived (for historical decryption) or deleted (for aggressive FS).

**3. Algorithm sunset list**

The LibertyChat core ships a signed, maintainer-updated list:

```
sunset_after = {
    "Ed25519-only":        2030-01-01,
    "ML-KEM-512":          2032-01-01,  # parameter too small
    "SHA-256-only-hash":   2035-01-01,
    ...
}
```

Clients refuse operations using the relevant suite after the sunset date. Updates to this list ship as signed update packages (separate from code updates), so even older clients without a fresh app build are warned about deprecated suites.

### Critical detail: defense against downgrade attacks

Cipher agility is a security risk **if implemented wrong**. The classical pitfall (TLS had this for years): an attacker manipulates ciphersuite negotiation and forces both sides onto the weakest mutually supported suite.

LibertyChat defense:

- The ciphersuite selection is included in the **MLS transcript hash**.
- Every Commit signs the entire negotiation history.
- A downgrade attempt changes the hash → signature verification fails.

### Why this is superior

- **Signal**: ciphersuite is hard-coded per protocol version. PQXDH was a **separate protocol version** rolled out as a migration. No in-band upgrade possible.
- **WhatsApp**: hard-coded; migrations are server-driven.
- **Matrix**: has ciphersuite fields but they are barely used; migrations happen via client updates in practice.
- **SimpleX**: hard-coded ciphersuite per protocol major version.
- **LibertyChat**: cipher agility is **structurally** anchored in the protocol, with downgrade defense, multi-suite KeyPackages, and automatic sunset enforcement. If ML-KEM falls in 5 years, the migration path is not "app update + re-pair all contacts" but "register a new suite, issue a Reinit Commit per group".

---

## Step 8: Key lifecycle and rotation

Hybrid PQ means **four instead of two** identity keys per device. Each has its own lifecycle. Managed badly → security gap. LibertyChat defines this explicitly.

### The four identity keys (per device)

| Key | Algorithm | Purpose | Lifetime | Rotation trigger |
|---|---|---|---|---|
| Sign-classical | Ed25519 | Classical signatures | up to device lifetime | compromise or PCS trigger |
| Sign-PQ | ML-DSA-65 | PQ signatures | up to device lifetime | compromise or PCS trigger |
| Encrypt-classical | X25519 | Classical HPKE | same as sign-classical | rotated with sign-key |
| Encrypt-PQ | ML-KEM-768 | PQ HPKE | same as sign-PQ | rotated with sign-key |

Convention: all four rotate **together**. Rotating one alone breaks the hybrid model (you would have a fresh half and an aged half whose compromise is no longer protected by the fresh half).

### KeyPackages — short-lived by design

KeyPackages are **single-use** and short-lived. Lifecycle:

1. **Generation**: device generates a batch of 50 KeyPackages at once (ephemeral HPKE keys + bundle signature with identity keys).
2. **Publish**: all 50 uploaded to the mailbox server.
3. **Consumption**: each invitation consumes exactly one (server deletes it on delivery).
4. **Refill**: as soon as the batch drops below 10, the device automatically generates a new batch.
5. **Expiry**: every KeyPackage carries a hard expiration date (e.g. 30 days). After that no client will accept an "old" KeyPackage even if it is still in the server pool.

The expiration date matters: without it, an attacker who eventually compromises the ephemeral HPKE private key gains **retroactive access** to all Welcome messages generated under that KeyPackage.

### Identity key rotation — scheduled and unscheduled

**Scheduled rotation (PCS for identity)**: every 12 months the device generates fresh identity keys, signs the new with the old (continuity chain), publishes a rotation statement. Every group the device is a member of receives the new identity key via an MLS Update Commit.

```
Continuity statement:
{
  old_pubkey_classical: ...,
  old_pubkey_pq: ...,
  new_pubkey_classical: ...,
  new_pubkey_pq: ...,
  rotation_timestamp: ...,
  signature_old_classical: ...,  // by old Ed25519
  signature_old_pq: ...,         // by old ML-DSA
  signature_new_classical: ...,  // by new Ed25519 (self-sign)
  signature_new_pq: ...,         // by new ML-DSA (self-sign)
}
```

Every counterparty can independently verify: "yes, this new key belongs to the identity I have trusted for 3 years."

**Unscheduled rotation (compromise)**: the device reports a compromise (or the user triggers manually). Immediately:

1. Generate new identity keys.
2. Invalidate all KeyPackages signed by the old keys (tombstone marker on the mailbox server).
3. Issue Update Commits in **every** group the device is a member of.
4. Publish new KeyPackages.
5. **Optional**: revocation statement signed by the old key ("this key is compromised, do not trust it") — distributed to all known contacts.

### Key material storage on the device

Four levels of sensitivity:

| Material | Stored where |
|---|---|
| Identity private keys (4) | **TEE / hardware keystore** (Secure Enclave, StrongBox, TPM) when available; otherwise OS keychain with passphrase |
| Active MLS epoch secrets | encrypted in app database, deleted on app sleep |
| KeyPackage HPKE private keys (for Welcome receipt) | app database, deleted on consumption |
| Per-message ratchet keys | RAM only, deleted after decryption |

Identity private keys never leave the device. Even at backup time only a **Shamir share** is exported (see Feature 12), never the full key.

### Why this is superior

- **Signal**: a single classical identity key, no defined rotation cycle. If your key is compromised you are effectively forced to create a new account (phone number reset).
- **WhatsApp**: identity tied to phone number, "rotation" = SIM swap.
- **Matrix**: cross-signing keys exist, but rotation has UX friction (re-verifying every contact).
- **SimpleX**: no global identity keys, only per-contact keys. Compromise of one contact breaks only that link. Good for pseudonymity, bad for "verified long-lived identity".
- **LibertyChat**: **structured 12-month rotation with a continuity chain**, atomic compromise response, hardware storage by default, all four hybrid keys managed together. No account swap needed at rotation; contacts verify automatically via the signature chain.

---

## Step 9: Trust anchors and verification of hybrid identity

Crypto alone is not enough. The user must be certain that the **public key they are encrypting to** actually belongs to the intended recipient. Otherwise even the strongest end-to-end encryption is moot — a man-in-the-middle attacker inserts their own key, the user encrypts to them, they re-encrypt to the real recipient.

With classical crypto this is already a problem. With hybrid it becomes more complex, because **two** keys must be verified.

### The verification problem, concretely

Alice adds Bob as a contact. She receives:

- `Bob_Ed25519_pub` (classical identity)
- `Bob_MLDSA_pub` (PQ identity)
- Both sign KeyPackages and membership updates.

Question: how does Alice know that both actually belong to Bob and not to a MitM?

LibertyChat offers **four verification mechanisms** with ascending strength.

### Mechanism 1: Safety Number (out-of-band, short form)

A deterministically derivable **safety number** is computed from all hybrid pubkeys:

```
safety_number = BLAKE3(
    "libertychat-safety-v1" ||
    sort([Alice_Ed25519_pub, Alice_MLDSA_pub,
          Bob_Ed25519_pub,   Bob_MLDSA_pub])
)[:30]  // 30 bytes = 60 hex characters
```

Critical: **all four hybrid keys go in**, both sides symmetric. Both see an identical safety number.

Displayed as twelve five-digit decimal groups (Signal-style):

```
12345 67890 23456 78901 34567 89012
45678 90123 56789 01234 67890 12345
```

Alice and Bob compare via an **out-of-band channel** (phone call, in-person meeting, video call). If the numbers match, no MitM sits in between.

### Mechanism 2: QR code (out-of-band, visual)

In person, Bob shows his full identity block as a QR code:

```
QR payload:
{
  v: 1,
  identity_classical: Bob_Ed25519_pub,
  identity_pq: Bob_MLDSA_pub,
  encrypt_classical: Bob_X25519_pub,
  encrypt_pq: Bob_MLKEM_pub,
  mailbox_endpoint: lvm://...,
  display_name: "Bob",
  signature_classical: ...,  // self-signed
  signature_pq: ...
}
```

Alice scans → her client verifies both self-signatures → identity is confirmed. Hybrid keys are validated together.

QR code size: ~3 KB payload → QR version 25 (thumb-sized). Scannable with a standard phone camera in under 2 seconds.

### Mechanism 3: TOFU with cryptographic continuity

"Trust On First Use" — on first contact, Alice simply trusts the keys she received. Once Bob's keys are pinned, **every future key change** is verified via the continuity chain (see Step 8):

```
old_keys → signed → new_keys
```

If the signature is valid, the rotation is legitimate. If it is not (no valid transition), the client raises a **security alert**:

```
⚠ Bob's key has changed.
   The new identity was NOT signed by the previous one.
   Possibly a new device — or an attacker.
   [Verify via QR / safety number]   [Block]
```

TOFU is weaker than out-of-band but practical for contacts you do not meet in person.

### Mechanism 4: Web-of-trust via signed endorsements (optional)

Alice can **endorse** Bob's identity by signing his identity bundle with her own keys:

```
Endorsement {
  endorser: Alice_Ed25519_pub + Alice_MLDSA_pub,
  endorsed: Bob_Ed25519_pub + Bob_MLDSA_pub,
  endorsed_display_name: "Bob",
  endorsement_timestamp: 2026-05-13,
  endorsement_strength: "in_person_verified",  // or "video_call", "tofu"
  signature_classical: ...,
  signature_pq: ...
}
```

Carol, who trusts Alice but does not yet know Bob, sees Alice's endorsement and can **transitively** build trust. An endorsement graph emerges (similar to GPG web-of-trust, but hybrid and with explicit strength markers).

Strictly optional. Anyone who does not want a web-of-trust simply ignores it. Anyone who does gets additional trust without a central authority.

### What happens on compromise

When a key is compromised and the user rotates (unscheduled, see Step 8):

- The continuity chain is broken, because the attacker had the old private keys and could sign a fake "new identity".
- Therefore: a **revocation statement** is published in parallel with the rotation, signed by the old key:

```
Revocation {
  revoked_keys: [old_Ed25519_pub, old_MLDSA_pub],
  revocation_reason: "compromise_suspected",
  revocation_timestamp: ...,
  new_identity_anchor: <new_pubkeys>,
  signatures: ...
}
```

Once contacts have seen the revocation statement, they **no longer accept continuity statements** from the old key. Re-verification via QR/safety number is required.

### Why this is superior

- **Signal**: safety numbers cover only the classical identity (no hybrid coverage). No web-of-trust. Continuity verification exists but only over the single identity key.
- **WhatsApp**: safety numbers present, but 99% of users never compare them. No web-of-trust.
- **Matrix**: cross-signing with complex device verification UX, frequent source of user confusion. No hybrid verification.
- **SimpleX**: SAS (short authentication string) for connection confirmation. No long-lived identity verification path, because no global identities exist.
- **Jami**: TOFU with DHT; no structured web-of-trust.
- **LibertyChat**: **four graduated verification mechanisms** with **hybrid coverage**, a clean continuity chain with compromise response, optional web-of-trust without a CA. The user picks strength by threat model — from "TOFU for family" to "in-person QR + endorsed by 3 trusted parties for activism".

---

## Step 10: Implementation and audit

The best crypto design is worthless if the implementation is broken. For hybrid PQ there are additional risks: two algorithms in parallel mean a doubled implementation surface, a doubled side-channel attack surface, and new classes of bugs (e.g. a wrongly implemented combiner).

LibertyChat strategy: **never reimplement what has already been vetted; always implement the workflow ourselves.**

### Foundation libraries (not written by us)

| Component | Library | Reason |
|---|---|---|
| MLS protocol | **mls-rs** (AWS) | Production-grade, deployed in AWS Wickr, external audit |
| X25519, Ed25519 | **dalek-cryptography** (`x25519-dalek`, `ed25519-dalek`) | Industry standard in the Rust ecosystem, peer-reviewed |
| ML-KEM-768 | **ml-kem** (RustCrypto WG) or **liboqs-rust** | RustCrypto variant: pure Rust, constant-time. liboqs: C bindings to NIST reference code |
| ML-DSA-65 | **ml-dsa** (RustCrypto) or **liboqs-rust** | Same choice |
| X-Wing combiner | **x-wing-rs** (in development, internal impl against IRTF draft) | Hybrid combiner per IRTF spec |
| HPKE | **hpke-rs** | RFC 9180-compliant, compatible with MLS |
| AEAD (ChaCha20-Poly1305 / AES-GCM) | **ring** or **RustCrypto** | both audited |
| HKDF | **hkdf** crate | trivially correct |
| BLAKE3 | **blake3** official crate | single vendor, but reference implementation |

**Deliberate choice per algorithm**: pure Rust where available (`x25519-dalek`, RustCrypto ML-KEM impl); C bindings only where unavoidable (`liboqs` as fallback while pure-Rust PQ implementations mature).

### What LibertyChat writes itself

- **Protocol layer**: mailbox queue, KeyPackage management, sealed sender, VRF queue rotation, etc.
- **Hybrid combiner wiring**: how KEM outputs flow into HKDF, how signatures are concatenated, how the ciphersuite ID is interpreted.
- **Continuity chain and revocation logic.**
- **Trust verification** (safety number, QR, TOFU pinning, WoT endorsements).
- **Cipher-agility negotiation** and downgrade defense.

### Test vectors

Three categories:

**1. NIST test vectors (mandatory)**

ML-KEM and ML-DSA ship with official NIST CAVP test vectors. The LibertyChat test suite runs them 1:1 and verifies byte-identity. If a dependency update introduces a regression, it is rejected immediately.

**2. MLS interop test vectors (from the IETF MLS WG)**

The MLS Working Group maintains a corpus of test vectors (Welcome messages, Commits, schedule outputs). LibertyChat and mls-rs are checked against it.

**3. LibertyChat hybrid test vectors (our own)**

We publish vectors for our LibertyChat-specific constructions:

- Hybrid combiner output for known inputs
- Ciphersuite selection hash for a given transcript
- Safety number computation for given identity bundles

Any re-implementer (e.g. in another language) can verify byte compatibility against these.

### Formal verification

Before v1.0 the protocol is formally modeled. Two parallel approaches:

**Tamarin Prover** (symbolic crypto, good for authentication properties)

Models for:

- Initial handshake with hybrid KEM
- KeyPackage replay resistance
- Compromise recovery (PCS properties)
- Ciphersuite negotiation against downgrade attacks

Expected output statements:

- "If both sides complete the handshake, they hold identical shared secrets" (authentication)
- "A passive attacker cannot compute the shared secret as long as either KEM is secure" (confidentiality under the hybrid assumption)
- "An active attacker cannot force a downgrade to a weaker suite" (downgrade resistance)

**ProVerif** (applied pi-calculus, good for equivalence properties)

Models for:

- Sealed-sender anonymity (server does not learn the sender)
- Forward secrecy after an epoch transition
- Post-compromise security after an Update Commit

### External audits

Pre-1.0 release gate:

1. **Crypto audit** by Trail of Bits, Cure53, or NCC Group. Scope:
   - Hybrid combiner implementation
   - KeyPackage lifecycle including replay resistance
   - Trust verification
   - Side-channel analysis (timing, power-analysis surface)

2. **Protocol audit** by an academic group (e.g. INRIA, NCC, or Real-World-Crypto Network). Scope:
   - Tamarin/ProVerif model review
   - Adversary model discussion
   - PCS and FS property validation

3. **Implementation fuzzing** as continuous integration:
   - `cargo-fuzz` on every parser (Welcome messages, KeyPackages, continuity statements)
   - libFuzzer with AFL++ as backend
   - Coverage > 85% for parsers/serializers

4. **Reproducible build verification**:
   - The audit firm builds from source and compares the resulting binary against the release binary
   - Nix or Bazel as the deterministic build system
   - An audit attestation is published with the release

### Bug bounty program

Pre-1.0: a private bounty with ~10 invited researchers (Real-World-Crypto community).
Post-1.0: a public bounty via HackerOne or managed directly. Reward tiers:

| Severity | Reward |
|---|---|
| Crypto break (KEM / signature / combiner) | $50,000 |
| Authentication bypass | $20,000 |
| Metadata leak (sealed sender, queue linking) | $10,000 |
| DoS against mailbox server | $2,000 |
| Minor issues | $200–$1,000 |

### Reference implementation strategy

`libertychat-core` is **the** reference implementation. All clients (desktop, mobile, CLI) are thin UI layers on top of it. Consequences:

- A core audit covers all clients.
- A crypto bug is fixed everywhere immediately.
- Native mobile bindings (Swift via UniFFI, Kotlin via JNI) reduce to pass-through.

No re-implementations in JavaScript or C++; anyone building an alternative client calls `libertychat-core` via FFI.

### Why this is superior

- **Signal**: open source, external audits (NCC 2016, Trail of Bits periodically), but **no formal model** for the newer hybrid extensions. Reproducible builds only partial (iOS impossible due to App Store signing).
- **WhatsApp**: closed source. Self-published whitepapers, no independent code audit possible.
- **Matrix**: open source, individual audits of Element/Olm, **no central formal correctness proof** across the entire system. Reproducible builds not end-to-end.
- **SimpleX**: open source, Trail of Bits audit in 2024, but **no formal model**. Build reproducibility unclear.
- **LibertyChat**: **three-tier audit gate** (crypto + protocol + reproducible-build verification), **formal Tamarin and ProVerif models** as a release prerequisite, **bug bounty with explicit tiers**, **reference implementation as single source of truth**. No other open-source messenger has all four as a hard release gate.

---

**Feature 2 (Hybrid Post-Quantum from Day One) is complete.** Walkthrough continues with feature 3.
