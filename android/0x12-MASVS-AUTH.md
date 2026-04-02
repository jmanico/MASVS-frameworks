# MASVS-AUTH: Authentication and Authorization

## Overview

Android apps authenticate users through remote protocols (OAuth 2.0, OpenID Connect, passkeys) and local mechanisms (BiometricPrompt, device credentials). Both paths have Android-specific implementation requirements. This category ensures authentication is implemented securely on the Android client — using the Credential Manager API, crypto-bound biometrics, and proper token management — while recognizing that server-side enforcement is the authoritative control.

## Android Authentication Architecture

```
┌────────────────────────────────────────────┐
│                  User                       │
│         ┌──────────┐                        │
│         │ Biometric│ ← Fingerprint/Face     │
│         └────┬─────┘                        │
└──────────────┼──────────────────────────────┘
               │
┌──────────────▼──────────────────────────────┐
│          BiometricPrompt                     │
│   (CryptoObject binding to KeyStore key)     │
│                                              │
│   ┌────────────────────────────────────┐     │
│   │      Credential Manager API        │     │
│   │  ┌──────┐ ┌────────┐ ┌─────────┐  │     │
│   │  │Passkey│ │Password│ │Federated│  │     │
│   │  │(FIDO2)│ │ (auto) │ │(Google) │  │     │
│   │  └──┬───┘ └───┬────┘ └────┬────┘  │     │
│   └─────┼─────────┼───────────┼────────┘     │
└─────────┼─────────┼───────────┼──────────────┘
          │         │           │
          ▼         ▼           ▼
    AndroidKeyStore  Server    Identity
    (TEE/StrongBox)            Provider
```

## Key Android APIs

- **Credential Manager** (`androidx.credentials`) — Unified API for passkeys, passwords, federated sign-in
- **BiometricPrompt** (`androidx.biometric`) — Biometric authentication with Class 3 (strong) support
- **Protected Confirmation** (`android.security.ConfirmationPrompt`) — TEE-signed user confirmation
- **AndroidKeyStore** — Hardware-backed keys bound to user authentication

## Controls Summary

| Control | Title |
|---|---|
| MASVS-AUTH-1 | The app uses secure authentication and authorization protocols and follows the relevant best practices |
| MASVS-AUTH-2 | The app performs local authentication securely according to the platform best practices |
| MASVS-AUTH-3 | The app secures sensitive operations with additional authentication |

## OWASP Mobile Top 10 2024 Mapping

- **M3 — Insecure Authentication/Authorization:** Directly addressed by all three controls
- **M1 — Improper Credential Usage:** Addressed by MASVS-AUTH-1.3, 1.4, 1.5

---

## MASVS-AUTH-1

### Control

The app uses secure authentication and authorization protocols and follows the relevant best practices.

### Description

Most Android apps connect to remote endpoints that require user authentication and enforce authorization. While server-side enforcement is critical, the Android client must also implement authentication flows securely — protecting tokens in transit and at rest, using platform-provided credential management, and avoiding implementation patterns that expose credentials to interception or replay. This control covers secure use of authentication protocols on the Android client side.

### Android Sub-Requirements

#### MASVS-AUTH-1.1 — Use the Credential Manager API for Authentication

The app uses `androidx.credentials.CredentialManager` as the unified authentication surface for passkeys (FIDO2 WebAuthn), passwords, and federated sign-in. The app does not use the deprecated FIDO2 library (`com.google.android.gms:play-services-fido`) or custom authentication UI for standard credential types.

**Rationale:** Credential Manager provides a single, consistent, and secure authentication flow that integrates with the platform's autofill, biometric prompt, and password manager infrastructure. It reduces implementation surface area and supports passkeys natively.

**Android References:**
- `androidx.credentials.CredentialManager`
- `CreatePublicKeyCredentialRequest` — passkey registration
- `GetPublicKeyCredentialOption` — passkey authentication
- `GetPasswordOption` — password retrieval

#### MASVS-AUTH-1.2 — Prefer Passkeys Over Passwords

The app supports passkeys (FIDO2 discoverable credentials) as the primary authentication mechanism. Passkeys are synced via Google Password Manager and provide phishing-resistant, password-free authentication.

**Rationale:** Passkeys eliminate password reuse, phishing, and credential stuffing attacks. On Android 9+, passkeys are hardware-backed (private key in AndroidKeyStore/TEE) and synced across devices.

#### MASVS-AUTH-1.3 — Store Authentication Tokens Securely

OAuth tokens, session tokens, JWTs, and API keys are stored in the AndroidKeyStore (encrypted with a hardware-backed key) or in encrypted DataStore/Tink storage. Tokens are never stored in plaintext SharedPreferences, databases without encryption, or external storage.

**Rationale:** Stolen tokens allow session hijacking and unauthorized API access. Hardware-backed encrypted storage ensures tokens survive only device compromise with root + TEE bypass.

#### MASVS-AUTH-1.4 — Implement Token Refresh and Expiration

The app implements short-lived access tokens with refresh token rotation. Expired tokens are not reused. The refresh token is stored with stronger protection (biometric-bound key) than the access token.

**Rationale:** Short-lived tokens limit the window of compromise. Refresh token rotation detects token theft (when both parties attempt to use the same refresh token).

#### MASVS-AUTH-1.5 — Clear Authentication State on Logout

On user logout, the app: 1. Revokes the refresh token server-side 2. Deletes all local tokens from encrypted storage 3. Clears any cached credentials from `CredentialManager` 4. Clears WebView cookies and session data if web-based auth was used 5. Navigates to the login screen and clears the back stack.

**Rationale:** Incomplete logout leaves residual authentication state that can be reused by subsequent users of the device or by malware.

#### MASVS-AUTH-1.6 — Do Not Implement Custom TLS or Certificate Validation in Auth Flows

Authentication flows use the platform's default `HttpsURLConnection` or `OkHttp` TLS implementation. The app does not implement custom `X509TrustManager`, `HostnameVerifier`, or `SSLSocketFactory` that weakens validation during authentication.

**Rationale:** Custom TLS implementations in authentication flows frequently introduce trust-all or hostname-skip vulnerabilities that enable credential interception via MITM.

---

## MASVS-AUTH-2

### Control

The app performs local authentication securely according to the platform best practices.

### Description

Many Android apps authenticate users locally via biometrics (fingerprint, face) or device credentials (PIN, pattern, password) using the BiometricPrompt API. Local authentication must be implemented correctly to prevent bypass — specifically, authentication results must be cryptographically bound to AndroidKeyStore operations rather than relying solely on callback success/failure booleans. This control ensures local authentication on Android is robust against instrumentation and tampering.

### Android Sub-Requirements

#### MASVS-AUTH-2.1 — Use BiometricPrompt with CryptoObject Binding

Local biometric authentication uses `BiometricPrompt.authenticate(CryptoObject, ...)` (crypto-bound mode), not `BiometricPrompt.authenticate(...)` (result-only mode). The `CryptoObject` wraps a `Cipher`, `Signature`, or `Mac` initialized with a key from the AndroidKeyStore that requires user authentication.

**Rationale:** Result-only (convenience) authentication returns a success/failure boolean in the `AuthenticationCallback`. On a compromised device, an attacker can hook the callback and force a success result. Crypto-bound authentication ties biometric success to an actual cryptographic operation — the key is only usable after successful biometric verification by the TEE, which cannot be spoofed in software.

**Android References:**
- `BiometricPrompt.authenticate(BiometricPrompt.CryptoObject, BiometricPrompt.PromptInfo)`
- `BiometricPrompt.CryptoObject(cipher)` — wraps an initialized `Cipher`
- Key requirement: `KeyGenParameterSpec.Builder.setUserAuthenticationRequired(true)`

#### MASVS-AUTH-2.2 — Require Strong Biometrics (Class 3)

The app requires `BiometricManager.Authenticators.BIOMETRIC_STRONG` (Class 3) for security-sensitive operations. Class 3 biometrics have hardware-backed security verification and meet the Android CDD's False Acceptance Rate requirements.

**Rationale:** `BIOMETRIC_WEAK` (Class 2) biometrics (e.g., some face unlock implementations) may not have hardware-backed matching and can be more easily spoofed. `BIOMETRIC_STRONG` ensures the biometric match is verified in the TEE/StrongBox.

**Android References:**
- `PromptInfo.Builder.setAllowedAuthenticators(BIOMETRIC_STRONG)`
- `PromptInfo.Builder.setAllowedAuthenticators(BIOMETRIC_STRONG | DEVICE_CREDENTIAL)` for fallback

#### MASVS-AUTH-2.3 — Handle Biometric Enrollment Changes

Keys used for biometric authentication are created with `setInvalidatedByBiometricEnrollment(true)` (default). The app handles `KeyPermanentlyInvalidatedException` by re-enrolling the user (re-authenticating with primary credentials and generating a new key).

**Rationale:** When a new biometric is enrolled, an attacker who has device access could add their own fingerprint. Invalidating keys on enrollment change ensures the new biometric cannot access previously protected data without re-authentication.

#### MASVS-AUTH-2.4 — Check Biometric Availability Before Prompting

The app checks `BiometricManager.canAuthenticate(authenticatorType)` before displaying the biometric prompt and handles all result codes: BIOMETRIC_SUCCESS — proceed with biometric auth, BIOMETRIC_ERROR_NONE_ENROLLED — guide user to enroll biometrics, BIOMETRIC_ERROR_NO_HARDWARE — fall back to device credential, BIOMETRIC_ERROR_HW_UNAVAILABLE — retry later or fall back, BIOMETRIC_ERROR_SECURITY_UPDATE_REQUIRED — prompt for security update

#### MASVS-AUTH-2.5 — Implement Lockout on Failed Biometric Attempts

The app relies on the platform's biometric lockout (automatic after 5 failed attempts, with exponential backoff). The app does not implement custom retry logic that could bypass platform lockout.

**Rationale:** The Android platform enforces biometric lockout in the TEE/framework level. Custom retry logic that resets counters or allows unlimited attempts weakens this protection.

#### MASVS-AUTH-2.6 — Use Protected Confirmation for High-Value Transactions

For the highest-value operations (financial transfers, legal agreements), the app uses `android.security.ConfirmationPrompt` (Protected Confirmation, API 28+) to obtain a TEE-signed confirmation that the user approved a specific action, even if the Android OS is fully compromised.

**Rationale:** Protected Confirmation renders the prompt in Trusted UI (TEE), signs the user's confirmation with a TEE-held key, and is verifiable server-side. This provides the strongest possible user confirmation on Android.

**Android References:**
- `ConfirmationPrompt.Builder(context).setPromptText(text).setExtraData(serverChallenge).build()`
- `ConfirmationCallback.onConfirmed(dataThatWasConfirmed)` — TEE-signed result
- Requires `FEATURE_TRUSTED_CONFIRMATION` system feature

---

## MASVS-AUTH-3

### Control

The app secures sensitive operations with additional authentication.

### Description

Certain operations within an app require additional authentication beyond the initial login — changing account settings, authorizing payments, accessing highly sensitive data, or performing irreversible actions. On Android, this step-up authentication can leverage BiometricPrompt, the Credential Manager, server-side re-authentication, or Protected Confirmation. This control ensures that sensitive operations are gated by appropriate additional authentication.

### Android Sub-Requirements

#### MASVS-AUTH-3.1 — Gate Sensitive Operations with Step-Up Authentication

The app requires re-authentication (biometric, device credential, or server-side) before performing sensitive operations including: Changing passwords or security settings, Modifying payment methods or making financial transactions, Exporting or deleting account data, Granting new device or session access, Viewing highly sensitive PII (SSN, full account numbers).

**Rationale:** The initial authentication may have been performed minutes or hours ago. Step-up authentication ensures the current user is authorized for the specific sensitive operation.

#### MASVS-AUTH-3.2 — Bind Step-Up Authentication to Cryptographic Operations

Step-up re-authentication uses crypto-bound BiometricPrompt (see MASVS-AUTH-2.1) where the cryptographic operation (e.g., signing a transaction request) can only complete after successful biometric verification. The server verifies the signed request.

**Rationale:** Without crypto-binding, a compromised app can bypass the step-up prompt and directly call the sensitive API. Crypto-binding ensures the server receives proof that biometric authentication occurred.

#### MASVS-AUTH-3.3 — Use Per-Use Key Authentication for Critical Operations

For the most critical operations, the AndroidKeyStore key requires per-use authentication (`setUserAuthenticationParameters(0, AUTH_BIOMETRIC_STRONG)` — zero timeout means every use requires authentication). The key cannot be used in a time-bounded window after a single authentication.

**Rationale:** Time-bounded authentication (e.g., 30-second window) allows multiple operations after a single biometric prompt. Per-use authentication ensures each critical operation individually requires biometric verification.

#### MASVS-AUTH-3.4 — Implement Server-Side Step-Up Verification

The server independently verifies that step-up authentication was performed before executing the sensitive operation. Client-side authentication gates alone are insufficient — the server must verify a cryptographic proof (signed challenge, attestation) or re-authenticate via a server-side mechanism (OTP, re-entered password).

**Rationale:** Client-side authentication checks can be bypassed by modifying the app, hooking callbacks, or directly calling API endpoints. Server-side verification is the authoritative enforcement point.

#### MASVS-AUTH-3.5 — Support Multi-Factor Authentication

For high-risk applications (financial, healthcare, enterprise), the app supports multi-factor authentication combining: Something the user knows (password/PIN), Something the user has (device with passkey, TOTP authenticator), Something the user is (biometric via BiometricPrompt).

**Rationale:** Single-factor authentication can be compromised through individual vector (stolen password, spoofed biometric). MFA requires compromise of multiple independent factors.

#### MASVS-AUTH-3.6 — Rate-Limit Step-Up Authentication Attempts

The app and server implement rate limiting and lockout for step-up authentication failures. After a configured number of failed step-up attempts, the session is terminated and full re-authentication is required.

**Rationale:** Without rate limiting, an attacker who has session access can brute-force the step-up authentication. Session termination after repeated failures limits this attack.

---

## Training-Aligned Requirements

Android-specific requirements for MASVS-AUTH-1, MASVS-AUTH-2, and MASVS-AUTH-3.

### MASVS-AUTH-1: Secure Authentication Protocols

#### AUTH-ANDROID-1.1: System Browser for OAuth

OAuth and OpenID Connect flows MUST use the system browser via Chrome Custom Tabs (`CustomTabsIntent`) or equivalent. Embedded `WebView` MUST NOT be used for OAuth flows per RFC 8252.

**Testable:** Verify OAuth login launches a Custom Tab (system browser), not a WebView. Inspect for `CustomTabsIntent.Builder` usage.

#### AUTH-ANDROID-1.2: PKCE Requirement

All OAuth 2.0 authorization code flows MUST use Proof Key for Code Exchange (PKCE) with S256 challenge method.

**Testable:** Capture OAuth authorization request. Verify `code_challenge` and `code_challenge_method=S256` parameters are present.

#### AUTH-ANDROID-1.3: Token Storage Security

Access tokens: stored in memory or encrypted storage, 5-15 minute lifetime. Refresh tokens: stored in Android Keystore, rotated on every use. Tokens MUST NOT be stored in `SharedPreferences`, query parameters, or logged.

**Testable:** Inspect storage locations for tokens. Verify refresh token rotation (new token issued on each refresh). Verify access token TTL.

#### AUTH-ANDROID-1.4: DPoP Token Binding

For high-security apps, access tokens SHOULD be sender-constrained using DPoP (RFC 9449). DPoP proof keys SHOULD be hardware-bound using Android Keystore.

**Testable:** Verify DPoP proof header in API requests. Verify proof key originates from Keystore.

#### AUTH-ANDROID-1.5: Server-Side Token Revocation on Logout

On logout, the app MUST call the server-side token revocation endpoint, clear all tokens from Keystore and encrypted storage, and clear WebView cookies and cache.

**Testable:** Capture logout network traffic for revocation call. Verify local token storage is empty post-logout. Verify WebView data cleared.

#### AUTH-ANDROID-1.6: Credential Manager for Passkeys

The app SHOULD use the Android Credential Manager API (available since Android 9 via Play Services) for passkey (FIDO2/WebAuthn) authentication. Passkey private keys MUST be stored in hardware (Keystore).

**Testable:** Verify `CredentialManager.create()` / `CredentialManager.get()` API usage. Verify WebAuthn registration/authentication flows complete successfully.

#### AUTH-ANDROID-1.7: Session Concurrency Controls

The app SHOULD implement concurrent session detection, limiting active sessions (e.g., max 3 devices) and notifying users via push notification when new sessions are created.

**Testable:** Authenticate on multiple devices beyond limit. Verify oldest session is invalidated or user is notified.

### MASVS-AUTH-2: Secure Local Authentication

#### AUTH-ANDROID-2.1: BiometricPrompt with Class 3 Biometrics

Local biometric authentication MUST use `BiometricPrompt` API with `BIOMETRIC_STRONG` (Class 3) authenticators. Deprecated APIs (`FingerprintManager`) MUST NOT be used.

**Testable:** Verify `BiometricPrompt` usage with `setAllowedAuthenticators(BIOMETRIC_STRONG)`. No `FingerprintManager` imports.

#### AUTH-ANDROID-2.2: CryptoObject Binding

Biometric authentication MUST bind to a cryptographic operation via `BiometricPrompt.CryptoObject`. The biometric MUST gate a Keystore key operation (sign, encrypt, decrypt), not merely a UI callback.

**Rationale:** Without CryptoObject binding, the biometric callback can be hooked and bypassed via Frida or similar tools. The biometric must gate a cryptographic operation, not just a UI flag.

**Testable:** Verify `BiometricPrompt.authenticate(CryptoObject, ...)` is called (not the overload without CryptoObject). Verify CryptoObject wraps a Keystore-backed Cipher, Signature, or Mac.

#### AUTH-ANDROID-2.3: Biometric Invalidation on Enrollment Change

Keys gated by biometric authentication MUST be invalidated when biometric enrollment changes (new fingerprint added). Use `setInvalidatedByBiometricEnrollment(true)` in `KeyGenParameterSpec`.

**Testable:** Add a new fingerprint. Attempt to use the biometric-gated key. Verify the key is invalidated and re-authentication is required.

### MASVS-AUTH-3: Additional Authentication for Sensitive Operations

#### AUTH-ANDROID-3.1: Step-Up Authentication

Sensitive operations (e.g., financial transactions, profile changes, password resets) MUST require step-up authentication via biometric confirmation, PIN re-entry, or MFA code even within an active session.

**Testable:** Attempt sensitive operation within active session. Verify additional authentication prompt is displayed before operation proceeds.

#### AUTH-ANDROID-3.2: Background Session Handling

When the app moves to background, it MUST clear sensitive data from memory and invalidate the session UI. The refresh token MAY be retained for session resumption, but the user MUST re-authenticate for sensitive operations after returning from background.

**Testable:** Background the app, return after configurable timeout. Verify sensitive screens require re-authentication.
