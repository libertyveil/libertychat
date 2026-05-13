# Feature 11: Hardware Key Support

> Walkthrough pending. Will be filled step-by-step in subsequent sessions.

## Concept summary

Master identity stored inside hardware-protected keystores:

- **YubiKey** (USB / NFC)
- **Apple Secure Enclave** (iOS / macOS)
- **Android StrongBox** (Pixel / Samsung)
- **TPM 2.0** (Linux / Windows)

The secret key never leaves the hardware boundary. Signing operations happen inside the secure element. Malware with root access on the device cannot exfiltrate the key — it can only request signatures while the device is unlocked.

## Why this is superior (preview)

- **Signal / WhatsApp / Matrix / SimpleX**: identity keys live in normal user-space storage. Malware with root can copy them and impersonate the user permanently.
- **LibertyChat**: phishing-resistant, malware-resistant identity binding. Standard pattern from FIDO/WebAuthn applied to messenger identity.
