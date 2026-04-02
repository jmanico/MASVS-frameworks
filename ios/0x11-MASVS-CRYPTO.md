# MASVS-CRYPTO: Cryptography

## Overview

iOS provides CryptoKit as the modern Swift-native cryptographic framework, the Secure Enclave for hardware-backed key operations, and CommonCrypto as the legacy C API. iOS 26 introduces quantum-secure TLS 1.3 with post-quantum key exchange enabled by default. Make sure the app uses strong cryptography with proper key management and hardware-backed keys where available.

## iOS Cryptographic Architecture

```
App Process
  CryptoKit          CommonCrypto
  (Swift)            (C API, legacy)
       \              /
        Secure Enclave
        (P256 ECDSA, key agreement)
        Keys never leave hardware
        Own boot ROM + AES engine
```

## Approved Algorithm Reference

| Use Case | Algorithm | iOS API |
|---|---|---|
| Symmetric encryption | AES-256-GCM | `CryptoKit.AES.GCM` |
| Symmetric encryption | ChaCha20-Poly1305 | `CryptoKit.ChaChaPoly` |
| Signing | ECDSA P-256 | `CryptoKit.P256.Signing` |
| Signing | ECDSA P-384 | `CryptoKit.P384.Signing` |
| Key agreement | ECDH P-256 | `CryptoKit.P256.KeyAgreement` |
| HMAC | HMAC-SHA256 | `CryptoKit.HMAC<SHA256>` |
| Hashing | SHA-256, SHA-384, SHA-512 | `CryptoKit.SHA256`, etc. |
| Key derivation | HKDF | `CryptoKit.HKDF<SHA256>` |
| Password hashing | Argon2id, PBKDF2 | CommonCrypto `CCKeyDerivationPBKDF` |
| Hardware signing | ECDSA P-256 | `SecureEnclave.P256.Signing` |
| PRNG | System CSPRNG | `SecRandomCopyBytes` or `CryptoKit.SymmetricKey(size:)` |

## OWASP Mobile Top 10 2024 Mapping

- **M10 - Insufficient Cryptography:** Directly addressed by both controls
- **M1 - Improper Credential Usage:** Addressed by MASVS-CRYPTO-1.4, MASVS-CRYPTO-2.1

---

## MASVS-CRYPTO-1

### Control

The app employs current strong cryptography and uses it according to industry best practices.

### Description

Cryptography protects user data on iOS where physical device access is a realistic threat. CryptoKit provides modern, safe-by-default APIs. CommonCrypto is the legacy C API that requires more careful usage. Make sure the app uses only strong, current algorithms with correct parameters.

### iOS Sub-Requirements

#### MASVS-CRYPTO-1.1 - Use CryptoKit for All New Cryptographic Operations

Use CryptoKit (`AES.GCM`, `ChaChaPoly`, `P256`, `SHA256`, `HMAC`, `HKDF`) for all new code. CommonCrypto is acceptable for legacy code but should be migrated.

**Rationale:** CryptoKit provides safe-by-default APIs that prevent common misuse (automatic nonce generation, authenticated encryption only).

#### MASVS-CRYPTO-1.2 - Use Authenticated Encryption Only

All symmetric encryption uses AES-GCM (`CryptoKit.AES.GCM`) or ChaCha20-Poly1305 (`CryptoKit.ChaChaPoly`). Never use ECB mode, unauthenticated CBC, or any mode without integrity protection.

**Rationale:** Encryption without authentication allows ciphertext manipulation and oracle attacks. CryptoKit only exposes authenticated modes.

#### MASVS-CRYPTO-1.3 - Generate Random Values with System CSPRNG

All security-sensitive random values use `SecRandomCopyBytes` or CryptoKit's built-in random generation (`SymmetricKey(size:)`, nonce generation). Never use `arc4random` or `drand48` for security purposes.

**Rationale:** CryptoKit and `SecRandomCopyBytes` use the system CSPRNG. Other random sources may not provide cryptographic-quality randomness.

#### MASVS-CRYPTO-1.4 - Do Not Hardcode Cryptographic Keys

The app does not contain hardcoded keys, passwords, or secrets in source code, plists, asset catalogs, or bundled files.

**Rationale:** Hardcoded keys are extractable from the IPA via class-dump, strings, or Hopper/Ghidra. Any secret in the app bundle should be considered public.

#### MASVS-CRYPTO-1.5 - Use Secure Enclave for Hardware-Backed Signing

Signing operations for sensitive data use `SecureEnclave.P256.Signing` so the private key is generated and used exclusively inside the Secure Enclave hardware.

**Rationale:** Secure Enclave keys cannot be extracted even on jailbroken devices. The private key material exists only in the dedicated coprocessor.

#### MASVS-CRYPTO-1.6 - Use Argon2id or High-Iteration PBKDF2 for Password-Derived Keys

When deriving keys from passwords, use Argon2id (preferred) or PBKDF2-HMAC-SHA256 with at least 600,000 iterations.

**Rationale:** Low iteration counts allow brute-force attacks on password-derived keys.

#### MASVS-CRYPTO-1.7 - Specify Full Algorithm Parameters

Every cryptographic operation specifies the complete algorithm configuration. Do not rely on default parameters from CommonCrypto that may select insecure options.

**Rationale:** CommonCrypto defaults can vary and may select insecure modes. CryptoKit avoids this problem with its type-safe API.

#### MASVS-CRYPTO-1.8 - Prohibited Algorithms

The app does not use DES, 3DES, RC4, Blowfish, MD5, SHA-1 for signatures, ECB mode, RSA with PKCS#1 v1.5 for encryption, or any custom/proprietary cipher.

---

## MASVS-CRYPTO-2

### Control

The app performs key management according to industry best practices.

### Description

Strong algorithms are useless with poor key management. iOS provides the Keychain for key storage, the Secure Enclave for hardware-bound keys, and App Attest for remote key verification. Make sure keys are generated, stored, rotated, and destroyed properly.

### iOS Sub-Requirements

#### MASVS-CRYPTO-2.1 - Generate Keys Inside the Secure Enclave When Possible

Signing and key agreement keys use `SecureEnclave.P256.Signing.PrivateKey()` or `SecureEnclave.P256.KeyAgreement.PrivateKey()` so key material never exists in app memory.

**Rationale:** Secure Enclave keys are non-extractable by design.

#### MASVS-CRYPTO-2.2 - Protect Keychain-Stored Keys with Access Control

Keys stored in the Keychain use `SecAccessControl` with `.biometryCurrentSet` (not `.biometryAny`) to require biometric authentication and invalidate on enrollment changes.

**Rationale:** `.biometryAny` allows new biometric enrollments to access existing keys. `.biometryCurrentSet` invalidates keys when biometrics change, preventing an attacker who adds their biometric from accessing protected data.

#### MASVS-CRYPTO-2.3 - Set Appropriate Keychain Accessibility Classes

Each Keychain item uses the most restrictive accessibility class that supports its use case:

- `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` for items needed only when the user is actively using the app.
- `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` for items needed in background processing.

**Rationale:** Overly permissive accessibility classes keep keys available when they should not be.

#### MASVS-CRYPTO-2.4 - Delete Keys on Logout and Account Deletion

The app deletes all Keychain items and Secure Enclave keys when the user logs out or deletes their account using `SecItemDelete`.

**Rationale:** Residual keys persist in the Keychain across app reinstalls (unlike the app container). Orphaned keys are unnecessary attack surface.

#### MASVS-CRYPTO-2.5 - Use Key Attestation via App Attest

For high-security use cases (payment, identity), the app uses `DCAppAttestService` to generate an attested key pair. The server validates the attestation object against Apple's CA to confirm the key was generated in genuine Apple hardware.

**Rationale:** App Attest provides server-verifiable proof that the app and device are genuine.

#### MASVS-CRYPTO-2.6 - iOS 26 Post-Quantum TLS

On iOS 26+, post-quantum TLS 1.3 key exchange is enabled by default. No developer action is needed for transport encryption. For application-layer encryption of long-lived data, evaluate HPKE or hybrid post-quantum schemes.

**Rationale:** Harvest-now-decrypt-later attacks threaten data encrypted today once quantum computers become practical.

#### MASVS-CRYPTO-2.7 - Never Export Secure Enclave Key Material

The app does not attempt to export or serialize Secure Enclave private keys. These keys are non-exportable by design.

**Rationale:** The security model depends on key material never leaving the hardware.

---

## Training-Aligned Requirements

iOS-specific requirements for MASVS-CRYPTO-1 and MASVS-CRYPTO-2.

### MASVS-CRYPTO-1: Strong Cryptography

#### CRYPTO-IOS-1.1: CryptoKit Usage

The app MUST use CryptoKit for all new cryptographic operations. CommonCrypto usage SHOULD be migrated.

**Testable:** Static analysis for CommonCrypto vs CryptoKit usage.

#### CRYPTO-IOS-1.2: Authenticated Encryption

All symmetric encryption MUST use authenticated modes (GCM or ChaChaPoly).

**Testable:** Verify no ECB or unauthenticated CBC in code.

#### CRYPTO-IOS-1.3: Secure Random

The app MUST use `SecRandomCopyBytes` or CryptoKit for security-sensitive random values.

**Testable:** Static analysis for non-CSPRNG random usage.

#### CRYPTO-IOS-1.4: No Hardcoded Secrets

The app MUST NOT contain hardcoded keys or secrets.

**Testable:** Run `strings` and `class-dump` on the IPA.

### MASVS-CRYPTO-2: Key Management

#### CRYPTO-IOS-2.1: Secure Enclave Key Generation

Signing keys for sensitive operations SHOULD use `SecureEnclave.P256`.

**Testable:** Verify CryptoKit Secure Enclave API usage.

#### CRYPTO-IOS-2.2: Keychain Access Control

Keys in Keychain MUST use `SecAccessControl` with `.biometryCurrentSet`.

**Testable:** Attempt key access after adding a new biometric. Verify invalidation.

#### CRYPTO-IOS-2.3: Key Lifecycle

Keys MUST be deleted on logout and account deletion.

**Testable:** Log out and inspect Keychain for residual items.

#### CRYPTO-IOS-2.4: App Attest Integration

High-security apps SHOULD use `DCAppAttestService` for key attestation.

**Testable:** Verify attestation token is sent to server and validated.
