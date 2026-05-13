# Feature 20: Reproducible Builds + Open Audit

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

- Every release binary is **deterministically buildable** from the published source — identical inputs produce byte-identical outputs.
- The build system uses Nix or Bazel for hermetic, reproducible builds.
- Pre-1.0 release is **gated by external audits** from Trail of Bits, Cure53, or equivalent.
- Formal protocol model in Tamarin or ProVerif before any production deployment.
- Open security bounty program.

The user can prove that the binary they install corresponds exactly to the source they (or auditors) have reviewed. No vendor backdoor can be silently inserted into release builds.

## Why this is superior (preview)

- **Signal**: reproducible builds for Android, partial for desktop; iOS impossible due to App Store signing.
- **WhatsApp / Telegram**: closed source, no reproducible builds.
- **Matrix (Element)**: open source but not reproducible by default.
- **LibertyChat**: reproducibility is a release gate. Source code, build recipe, and binary are independently verifiable.
