# MyCloud

**The private cloud for people who mean it.**

MyCloud is the shared infrastructure behind the MyPhone product family — identity, communication, storage, and software distribution for every device in the ecosystem. It is built on a single architectural principle: the server cannot read what it stores.

---

## What MyCloud Is

MyCloud is a device-independent cloud platform. It is not tied to any single product. Every device in the family — whatever you own, however many you own — connects to the same infrastructure with the same level of service.

| Service           | What it does                                                                                                                                                                                                  |
|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Identity**      | Issues cryptographic device certificates to every enrolled device via the MyCloud Device Provisioning CA. Devices trust each other because they share a root of trust, not because they were manually paired. |
| **Communication** | Operates a Matrix Conduit homeserver at matrix.mycloud.com. End-to-end encrypted messaging and video calls. The server forwards ciphertext — it cannot read your messages or see your calls.                  |
| **Storage**       | Encrypted media sync, device backup, and settings backup. Every file is encrypted on your device before it leaves. MyCloud stores opaque blobs.                                                               |
| **Distribution**  | Signs and delivers firmware updates for all devices via a CDN backed by a hardware HSM with multi-party quorum signing. No single person can produce a valid update.                                          |
| **Gaming relay**  | Routes RetroArch Netplay sessions between devices over the internet. Forwards encrypted packets only — no session data retained.                                                                              |
| **App Store**     | Reviews, signs, and distributes apps for all devices in the family. Three-stage security review pipeline before any package is distributed.                                                                   |

---

## Server-Blind by Architecture

The server cannot read your data. This is not a privacy policy and it is not a promise. It is an architectural property enforced by how the cryptography works.

**Two-layer encryption model:**

- **Layer A — MyCloud Account Keys** cover content you choose to share across your devices: photos, videos, playlists, documents, message history, and device settings. The Account Key is derived during account setup and shared only with your enrolled devices. MyCloud stores the encrypted key blob and the encrypted content. It holds neither the key nor the plaintext.

- **Layer B — Device-personal keys** cover secrets that belong to a single device and no other: wallet keys, biometric templates, device-specific backup. These keys never leave the device hardware. MyCloud has no access to them, no path to them, and no record of them.

The result: compromising MyCloud infrastructure yields zero plaintext user data. There is nothing to read.

---

## Key Infrastructure

### MyCloud Root CA

The root of cryptographic trust for the entire product family lives in a Thales Luna Network HSM 7 (FIPS 140-3 Level 3). It never leaves the HSM. All subordinate keys are generated in a formal ceremony with all quorum members physically present.

```
MyCloud Root CA  [Thales Luna HSM — never exported]
  ├── MyPhone OTA Firmware Signing Key    [3-of-5 quorum]
  ├── MyTV OTA Firmware Signing Key       [2-of-3 quorum]
  ├── MyCloud Device Provisioning CA      [2-of-5 quorum]
  │     ├── Device certs: MP-* (MyPhone family)
  │     └── Device certs: MTV-* (MyTV family)
  ├── MyCloud App Store Signing Key       [2-of-5 quorum]
  └── Socure Relay Key                    [1-of-5 quorum]
```

Each OTA signing key has an explicit authority boundary. The MyPhone key cannot sign MyTV firmware. The MyTV key cannot sign MyPhone firmware. Cross-signing is structurally impossible.

### Matrix Homeserver

MyCloud operates a maintained fork of [Conduit](https://conduit.rs) — a Rust-native Matrix homeserver at approximately 50K lines of code. We selected it for the same reason we make every infrastructure decision: minimal, auditable, memory-safe.

Privacy modifications applied to our fork:

- Zero-day message retention — routed E2EE messages are forwarded immediately and not stored
- Room membership history not persisted beyond routing requirements
- Access logs contain only hashed device serials, never plaintext identities
- Log retention: 24 hours maximum, then purged
- All device connections require TLS 1.3 with mutual certificate authentication — unauthenticated connections are refused at the handshake

### OTA Signing

Every firmware update for every device is signed by a key that lives in the Thales Luna HSM. The CDN receives signed packages. It cannot modify them, re-sign them, or produce new ones. Signing requires a quorum of keyholders physically present — no single person can produce a valid update under any circumstance.

---

## Device Identity

Every device in the product family receives a certificate from the MyCloud Device Provisioning CA at first boot, issued via hardware attestation.

Devices prove genuine hardware before receiving a certificate. A counterfeit device that cannot produce a valid attestation quote from its secure element will not be provisioned.

Device certificate namespaces:

- `MP-*` — issued to devices in the MyPhone family
- `MTV-*` — issued to devices in the MyTV family

Devices trust each other because they share this root of trust. No manual certificate pinning required. Revocation propagates via OCSP and delta CRL within 60 seconds of a loss report.

---

## App Store

The MyCloud App Store distributes apps for all devices in the family. Every submission passes three sequential review stages before receiving a signature.

| Stage | Process | Blocking criteria |
|---|---|---|
| **1 — Static analysis** | Automated, ~15 min. Banned syscall patterns, dependency vulnerability scan, manifest validation. | Any HIGH/CRITICAL CVE in critical dependencies. Declared permissions inconsistent with function. Cryptominer signatures. |
| **2 — Sandbox execution** | Automated, ~30 min. App runs in isolated container. Network traffic logged. Syscall audit. | Undeclared network destinations. Filesystem access outside declared scope. Any attempt to probe secure hardware interfaces. |
| **3 — Human review** | MyCloud security team, 5–10 business days. Privacy policy review, functionality verification, policy compliance. | Missing or contradictory privacy policy. Spyware or undisclosed data harvesting. |

**Developer terms:** 80% revenue share to developer / 20% to MyCloud. No proprietary SDK required. Any app packaged for the target platform's standard format is eligible.

---

## Subscription

MyCloud is offered at a single flat rate. No tiers. No per-device fees. No usage caps. One subscription covers every device you enroll.

The principle: anything that works over a local network between your own devices is free. The subscription covers what requires MyCloud infrastructure — messaging, call relay, cloud storage, Netplay relay, App Store access, and CDN-delivered updates.

---

## What MyCloud Does Not Do

- **No telemetry.** MyCloud collects no usage data, no location data, and no behavioural analytics from any device.
- **No content moderation.** MyCloud cannot read user content and therefore cannot moderate it. Abuse prevention operates at the account level via identity-verified device activation — not content inspection.
- **No key escrow.** Device-personal keys never reach MyCloud. There is no recovery path through MyCloud for Layer B secrets.

---

## Security Boundaries

These are constraints, not guidelines. Any proposed change that violates one requires a formal architectural review and a revision to this document.

| Boundary                          | Rule                                                                                                                                                                                |
|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Server blindness**              | MyCloud servers never hold plaintext user content at any point in any code path. Client-side encryption before upload is mandatory.                                                 |
| **Signing key isolation**         | OTA signing keys never leave the Thales Luna HSM. No single person can produce a valid firmware signature. Quorum policy is enforced in hardware.                                   |
| **OTA cross-signing prohibition** | Each device family's OTA key signs only that family's firmware. The key hierarchy makes cross-signing structurally impossible, not just policy-prohibited.                          |
| **Device class scope**            | Device certificate namespace (MP- vs MTV-) determines authorization at all MyCloud service endpoints. This is enforced server-side — a device cannot self-promote its access level. |

---
