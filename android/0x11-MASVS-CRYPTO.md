# MASVS-CRYPTO: Cryptography

## Overview

Cryptography is fundamental to protecting user data on Android, where physical device access is a realistic threat. Android provides the AndroidKeyStore for hardware-backed key storage, supports modern algorithms through `java.security` and `javax.crypto`, and offers Google Tink as a high-level cryptographic library. Make sure the app uses strong cryptography with proper key management, using Android's hardware security capabilities.

## Android Cryptographic Architecture

```
┌─────────────────────────────────────────┐
│            App Process                   │
│  ┌──────────┐  ┌───────────┐            │
│  │  Tink    │  │ java.     │            │
│  │  AEAD    │  │ security  │            │
│  └────┬─────┘  └─────┬─────┘            │
│       │               │                  │
│       └───────┬───────┘                  │
│               │                          │
│    ┌──────────▼──────────┐               │
│    │   AndroidKeyStore   │               │
│    │     Provider        │               │
│    └──────────┬──────────┘               │
└───────────────┼──────────────────────────┘
                │ Binder IPC
┌───────────────▼──────────────────────────┐
│         Keystore2 Daemon                 │
│    (keystore2, android.security)         │
└───────────────┬──────────────────────────┘
                │ HAL
┌───────────────▼──────────────────────────┐
│     Trusted Execution Environment        │
│  ┌────────────┐  ┌───────────────┐       │
│  │    TEE     │  │   StrongBox   │       │
│  │  (ARM      │  │  (Dedicated   │       │
│  │  TrustZone)│  │   Secure      │       │
│  │            │  │   Element)    │       │
│  └────────────┘  └───────────────┘       │
└──────────────────────────────────────────┘
```

## Approved Algorithm Reference

| Use Case | Algorithm | Android API |
|---|---|---|
| Symmetric encryption | AES-256-GCM | `KeyProperties.KEY_ALGORITHM_AES` + `BLOCK_MODE_GCM` |
| Asymmetric encryption | RSA-2048+ OAEP | `KEY_ALGORITHM_RSA` + `ENCRYPTION_PADDING_RSA_OAEP` |
| Signing | ECDSA P-256 | `KEY_ALGORITHM_EC` + `DIGEST_SHA256` |
| Signing | RSA-2048+ PSS | `KEY_ALGORITHM_RSA` + `SIGNATURE_PADDING_RSA_PSS` |
| HMAC | HMAC-SHA256 | `KEY_ALGORITHM_HMAC_SHA256` |
| Key agreement | ECDH P-256 | `KEY_ALGORITHM_EC` + `PURPOSE_AGREE_KEY` |
| PRNG | SecureRandom | `java.security.SecureRandom` (backed by `/dev/urandom`) |

## OWASP Mobile Top 10 2024 Mapping

- **M10 - Insufficient Cryptography:** Directly addressed by both controls
- **M1 - Improper Credential Usage:** Addressed by MASVS-CRYPTO-1.4, MASVS-CRYPTO-2.1

---

## MASVS-CRYPTO-1

### Control

The app employs current strong cryptography and uses it according to industry best practices.

### Description

Cryptography is critical for protecting user data on Android, where physical access to the device is a realistic threat model. Android provides platform cryptographic APIs (`java.security`, `javax.crypto`) and hardware-backed cryptographic operations via the AndroidKeyStore. Make sure the app uses only strong, current algorithms with correct parameters and avoids the many pitfalls that lead to weak or broken cryptographic implementations on Android.

### Android Sub-Requirements

#### MASVS-CRYPTO-1.1 - Use Approved Cryptographic Algorithms Only

The app uses only the following approved algorithms and modes:

| Use Case | Algorithm | Configuration |
|---|---|---|
| Symmetric encryption | AES-256 | GCM mode, no padding (`KeyProperties.BLOCK_MODE_GCM`, `ENCRYPTION_PADDING_NONE`) |
| Asymmetric encryption | RSA-2048+ | OAEP padding with SHA-256 (`ENCRYPTION_PADDING_RSA_OAEP`) |
| Digital signatures | ECDSA with P-256 | SHA-256 digest (`DIGEST_SHA256`) |
| Digital signatures | RSA-2048+ | PSS padding with SHA-256 (`SIGNATURE_PADDING_RSA_PSS`) |
| HMAC | HMAC-SHA256 or HMAC-SHA512 | `KEY_ALGORITHM_HMAC_SHA256` |
| Hashing (non-security) | SHA-256 or SHA-512 | `MessageDigest.getInstance("SHA-256")` |
| Key agreement | ECDH P-256 | `KEY_ALGORITHM_EC`, `PURPOSE_AGREE_KEY` |
| Password hashing | Argon2id or PBKDF2-HMAC-SHA256 | Minimum 600,000 iterations for PBKDF2 |

**Prohibited:** DES, 3DES, RC4, Blowfish, MD5, SHA-1 for signatures, ECB block mode, RSA with PKCS#1 v1.5 padding for encryption, any custom/proprietary cipher.

#### MASVS-CRYPTO-1.2 - Never Use ECB Block Mode

The app never uses AES in ECB mode (`KeyProperties.BLOCK_MODE_ECB`). ECB encrypts identical plaintext blocks to identical ciphertext blocks, leaking patterns.

**Rationale:** ECB mode is the default in many `Cipher.getInstance()` calls when no mode is specified (provider-dependent). Always specify the full transformation string: `"AES/GCM/NoPadding"`.

#### MASVS-CRYPTO-1.3 - Generate IVs and Nonces Securely

All initialization vectors (IVs) and nonces are generated using `SecureRandom` (which delegates to the kernel CSPRNG on Android). IVs are never hardcoded, reused across encryptions, or derived deterministically from predictable input.

**Rationale:** IV/nonce reuse in GCM mode is catastrophic because it enables key recovery. AES-GCM requires a unique 96-bit nonce for every encryption operation with the same key.

**Android References:**
- `SecureRandom` - backed by `/dev/urandom` on Android
- For AndroidKeyStore: the Keystore provider generates IVs internally; retrieve via `Cipher.getIV()` after `init()`

#### MASVS-CRYPTO-1.4 - Do Not Hardcode Cryptographic Keys

The app does not contain hardcoded cryptographic keys, passwords, or secrets in source code, XML resources, `strings.xml`, `BuildConfig` fields, native libraries (`.so` files), or the `assets/` directory.

**Rationale:** Hardcoded keys are extractable from the APK via `apktool`, `jadx`, or `strings` on native libraries. Any secret embedded in the APK should be considered public.

#### MASVS-CRYPTO-1.5 - Use the AndroidKeyStore Provider for Cryptographic Operations

All cryptographic operations on sensitive data use keys stored in the AndroidKeyStore (`KeyGenerator.getInstance(algorithm, "AndroidKeyStore")` or `KeyPairGenerator.getInstance(algorithm, "AndroidKeyStore")`).

**Rationale:** AndroidKeyStore keys are bound to the device's TEE or StrongBox secure element. The key material never enters the app's process memory and cannot be extracted, even on rooted devices (for hardware-backed keys).

**Android References:**
- `KeyGenerator.getInstance("AES", "AndroidKeyStore")`
- `KeyPairGenerator.getInstance("EC", "AndroidKeyStore")`
- `KeyStore.getInstance("AndroidKeyStore")`

#### MASVS-CRYPTO-1.6 - Verify Hardware-Backed Key Storage

The app verifies that cryptographic keys are hardware-backed by checking `KeyInfo.getSecurityLevel()` returns `SECURITY_LEVEL_TRUSTED_ENVIRONMENT` or `SECURITY_LEVEL_STRONGBOX`. If hardware backing is unavailable, the app degrades gracefully with appropriate risk acceptance.

**Android References:**
- `KeyFactory.getInstance(key.getAlgorithm(), "AndroidKeyStore").getKeySpec(key, KeyInfo.class)`
- `KeyInfo.getSecurityLevel()` (API 31+)
- `KeyInfo.isInsideSecureHardware()` (deprecated but available on older APIs)

#### MASVS-CRYPTO-1.7 - Use SecureRandom for All Random Number Generation

All security-sensitive random values (session tokens, nonces, salts, key material) are generated using `java.security.SecureRandom`. The app does not use `java.util.Random`, `Math.random()`, or `System.currentTimeMillis()` for security-sensitive purposes.

**Rationale:** `java.util.Random` uses a predictable linear congruential generator. `SecureRandom` on Android delegates to the kernel CSPRNG (`/dev/urandom`).

#### MASVS-CRYPTO-1.8 - Avoid Insecure Cipher.getInstance() Calls

Every `Cipher.getInstance()` call specifies the full transformation string including algorithm, mode, and padding (e.g., `"AES/GCM/NoPadding"`). The app never calls `Cipher.getInstance("AES")` without specifying mode and padding, as the default varies by provider and may select ECB.

**Rationale:** The default transformation for `Cipher.getInstance("AES")` is provider-dependent. On some Android versions and providers, it defaults to `AES/ECB/PKCS5Padding`, which is insecure.

---

## MASVS-CRYPTO-2

### Control

The app performs key management according to industry best practices.

### Description

Even strong cryptographic algorithms are compromised by poor key management. On Android, the platform provides the AndroidKeyStore system, hardware-backed key storage (TEE and StrongBox), and key attestation for remote verification. Make sure cryptographic keys are properly generated, stored, rotated, and destroyed throughout their lifecycle using Android-specific key management facilities.

### Android Sub-Requirements

#### MASVS-CRYPTO-2.1 - Generate Keys Inside the AndroidKeyStore

Symmetric and asymmetric keys are generated directly inside the AndroidKeyStore using `KeyGenerator` or `KeyPairGenerator` with the `"AndroidKeyStore"` provider, not generated externally and then imported. Make sure the key material never exists in app process memory.

**Android References:**
- `KeyGenerator.getInstance("AES", "AndroidKeyStore")` with `KeyGenParameterSpec`
- `KeyPairGenerator.getInstance("EC", "AndroidKeyStore")` with `KeyGenParameterSpec`

#### MASVS-CRYPTO-2.2 - Restrict Key Purpose

Each key is generated with the minimum necessary purposes via `KeyGenParameterSpec.Builder(alias, purpose)`. Encryption keys use `PURPOSE_ENCRYPT | PURPOSE_DECRYPT`. Signing keys use `PURPOSE_SIGN | PURPOSE_VERIFY`. Keys are never generated with `PURPOSE_ENCRYPT | PURPOSE_SIGN` combined.

**Rationale:** Multi-purpose keys increase the attack surface. A key used for both encryption and signing may be vulnerable to cross-protocol attacks.

#### MASVS-CRYPTO-2.3 - Bind Keys to User Authentication

Keys protecting high-value data require user authentication before use. The app configures:
- `setUserAuthenticationRequired(true)`
- `setUserAuthenticationParameters(timeoutSeconds, AUTH_BIOMETRIC_STRONG)` or `AUTH_BIOMETRIC_STRONG | AUTH_DEVICE_CREDENTIAL`
- Appropriate timeout (0 for per-use authentication, or a short duration for time-bounded access)

**Rationale:** Without authentication binding, any code running as the app's UID can use the key. Authentication binding ensures the user is physically present.

#### MASVS-CRYPTO-2.4 - Set Key Validity Periods

Keys have defined validity periods using `setKeyValidityStart()` and `setKeyValidityEnd()` or `setKeyValidityForOriginationEnd()` / `setKeyValidityForConsumptionEnd()` to enforce key rotation schedules.

**Rationale:** Unbounded key lifetimes increase the window for key compromise. Expired keys force the application to generate fresh key material.

#### MASVS-CRYPTO-2.5 - Invalidate Keys on Biometric Enrollment Changes

Keys bound to biometric authentication are configured with `setInvalidatedByBiometricEnrollment(true)` (the default) so that adding new fingerprints or face enrollments invalidates existing keys.

**Rationale:** If a new biometric is enrolled (potentially by an attacker who has device access), previously created keys should not be usable with the new biometric.

#### MASVS-CRYPTO-2.6 - Use Key Attestation for Remote Verification

For high-security use cases (e.g., payment, identity), the app generates a key pair with an attestation challenge from the server and sends the attestation certificate chain for server-side verification. The server:
1. Verifies the certificate chain signature
2. Confirms the root certificate matches the Google attestation root
3. Checks the `attestationSecurityLevel` is `TrustedEnvironment` or `StrongBox`
4. Validates the attestation extension for key properties
5. Checks certificate revocation status

**Android References:**
- `KeyGenParameterSpec.Builder.setAttestationChallenge(serverNonce)`
- Attestation certificate chain: `KeyStore.getCertificateChain(alias)`
- Google attestation root certificates (ECDSA P-384 root rotation begins February 2026)

#### MASVS-CRYPTO-2.7 - Delete Keys When No Longer Needed

The app deletes cryptographic keys from the AndroidKeyStore when they are no longer needed (e.g., user logout, account deletion, key rotation) using `KeyStore.deleteEntry(alias)`.

**Rationale:** Residual keys in the KeyStore persist across app updates and reinstalls (if the app is not uninstalled). Orphaned keys are unnecessary attack surface.

#### MASVS-CRYPTO-2.8 - Never Export Key Material from Hardware

The app does not attempt to export private or secret key material from hardware-backed storage. Keys generated in the AndroidKeyStore with `setIsStrongBoxBacked(true)` or hardware TEE backing are non-exportable by design.

**Rationale:** The entire security model of hardware-backed keys depends on the key material never leaving the secure element. Attempting to export (or wrapping with a software key) defeats this protection.

---

## Training-Aligned Requirements

Android-specific requirements for MASVS-CRYPTO-1 and MASVS-CRYPTO-2.

### MASVS-CRYPTO-1: Strong Cryptography

#### CRYPTO-ANDROID-1.1: Approved Algorithms Only

The app MUST use only current, strong cryptographic algorithms:
- **Symmetric encryption:** AES-256-GCM or ChaCha20-Poly1305
- **Hashing:** SHA-256, SHA-384, SHA-512, or SHA-3
- **Key derivation:** HKDF, PBKDF2 (with sufficient iterations), or Argon2
- **Asymmetric:** ECDSA (P-256 or P-384), RSA (2048+ bits with OAEP), Ed25519
- **HMAC:** HMAC-SHA256 or higher

The app MUST NOT use deprecated algorithms: DES, 3DES, RC4, MD5, SHA-1 (for security purposes), or Blowfish.

**Testable:** Static analysis identifies all cryptographic algorithm usage. No deprecated algorithms present.

#### CRYPTO-ANDROID-1.2: Authenticated Encryption

All symmetric encryption MUST use authenticated encryption modes (GCM, CCM, or Poly1305). ECB mode MUST NOT be used. CBC mode without authentication MUST NOT be used for new implementations.

**Testable:** Static analysis verifies all `Cipher.getInstance()` calls use authenticated modes. No ECB or unauthenticated CBC usage.

#### CRYPTO-ANDROID-1.3: Secure Random Number Generation

The app MUST use `java.security.SecureRandom` for all cryptographic random number generation. `java.util.Random`, `Math.random()`, or other non-cryptographic PRNGs MUST NOT be used for security-sensitive operations.

**Testable:** Static analysis confirms all security-related random generation uses `SecureRandom`.

#### CRYPTO-ANDROID-1.4: Keystore-Supported Algorithms

When cryptographic operations involve Android Keystore, the app MUST use Keystore-supported algorithms: AES-GCM, HMAC-SHA256, ECDSA, or RSA. Key operations SHOULD remain within the Keystore (TEE/StrongBox) without exporting key material.

**Testable:** Verify key operations use `KeyStore.getInstance("AndroidKeyStore")` and key material is not extracted.

#### CRYPTO-ANDROID-1.5: Post-Quantum Readiness

On Android 17+ (API 37), the app SHOULD evaluate use of HPKE hybrid cryptography (RFC 9180) for forward-looking quantum resistance where supported by the platform SPI.

**Testable:** Verify awareness and adoption plan for hybrid post-quantum key exchange on supported API levels.

### MASVS-CRYPTO-2: Key Management

#### CRYPTO-ANDROID-2.1: No Hardcoded Keys

The app MUST NOT contain hardcoded encryption keys, API keys, or secrets in source code, resources, assets, or native libraries.

**Testable:** Static analysis (grep, semgrep, or equivalent) finds no hardcoded keys or secrets in APK contents.

#### CRYPTO-ANDROID-2.2: Key Generation in Hardware

Cryptographic keys MUST be generated inside Android Keystore (TEE or StrongBox), not imported from software. Make sure key material never exists in app process memory.

**Testable:** Verify `KeyGenParameterSpec.Builder` is used with `setKeySize()` and key is generated via `KeyGenerator` or `KeyPairGenerator` with "AndroidKeyStore" provider.

#### CRYPTO-ANDROID-2.3: Key Attestation

For high-assurance scenarios, the app SHOULD use Android Key Attestation to allow the server to verify that keys were generated in hardware and have specific security properties.

**Testable:** Verify attestation certificate chain is sent to server and validated against Google's root certificate.

#### CRYPTO-ANDROID-2.4: Biometric-Gated Key Access

Keys protecting sensitive operations MUST be configured to require user authentication (biometric or device credential) before use, via `setUserAuthenticationRequired(true)` in `KeyGenParameterSpec`.

**Testable:** Attempt to use protected key without authentication. Verify `UserNotAuthenticatedException` is thrown.

#### CRYPTO-ANDROID-2.5: Key Lifecycle Management

The app MUST implement proper key rotation procedures. Keys MUST have defined validity periods set via `setKeyValidityStart()` and `setKeyValidityEnd()` where appropriate. Expired keys MUST NOT be used for new encryption operations.

**Testable:** Verify key validity parameters are configured. Verify expired keys are rejected for new operations.

#### CRYPTO-ANDROID-2.6: Symmetric Key Protection

If symmetric encryption keys must exist outside of Keystore (e.g., for database encryption), they MUST be wrapped (encrypted) by a Keystore-held key. The wrapping key MUST require user authentication.

**Testable:** Verify database encryption keys are wrapped by Keystore-held master keys. Verify wrapping key has authentication requirement.
