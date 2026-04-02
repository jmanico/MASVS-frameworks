# MASVS-AUTH: Authentication and Authorization

## Overview

iOS authenticates users through remote protocols (OAuth 2.0, OpenID Connect, passkeys) and local mechanisms (Face ID/Touch ID via LAContext). Both paths have iOS-specific requirements. Authentication must be implemented securely using ASWebAuthenticationSession, passkeys via AuthenticationServices, crypto-bound biometrics, and proper token management, while recognizing server-side enforcement as the authoritative control.

## iOS Authentication Architecture

```
User
  Face ID / Touch ID
       |
  LAContext + Keychain
  (SecAccessControl with .biometryCurrentSet)
       |
  AuthenticationServices
    Passkey    Password    Federated
    (FIDO2)    (AutoFill)  (Sign in with Apple)
       |          |            |
  Secure       Server      Identity
  Enclave                  Provider
```

## Key iOS APIs

- **AuthenticationServices** - Passkeys, AutoFill, Sign in with Apple
- **LocalAuthentication (LAContext)** - Face ID/Touch ID with Keychain integration
- **ASWebAuthenticationSession** - System browser for OAuth flows
- **Keychain with SecAccessControl** - Biometric-gated credential storage

## OWASP Mobile Top 10 2024 Mapping

- **M3 - Insecure Authentication/Authorization:** Directly addressed by MASVS-AUTH-1, MASVS-AUTH-2, MASVS-AUTH-3
- **M1 - Improper Credential Usage:** Addressed by MASVS-AUTH-1.3, 1.4, 1.5

---

## MASVS-AUTH-1

### Control

The app uses secure authentication and authorization protocols and follows the relevant best practices.

### Description

iOS apps connecting to remote endpoints must implement authentication flows securely by protecting tokens, using platform-provided credential management, and avoiding patterns that expose credentials. This covers secure use of authentication protocols on the iOS client.

### iOS Sub-Requirements

#### MASVS-AUTH-1.1 - Use ASWebAuthenticationSession for OAuth

OAuth and OpenID Connect flows MUST use `ASWebAuthenticationSession`, not `WKWebView` or `UIWebView`. Set `prefersEphemeralWebBrowserSession` for private sessions that do not share Safari cookies.

**Rationale:** ASWebAuthenticationSession runs in a separate process, shares Safari cookies for SSO, and prevents the app from intercepting credentials. WKWebView lets the app access all content including typed passwords.

**iOS Refs:** `ASWebAuthenticationSession`, `prefersEphemeralWebBrowserSession`

#### MASVS-AUTH-1.2 - Require PKCE for All OAuth Flows

All OAuth 2.0 authorization code flows MUST use PKCE with S256 challenge method. The code verifier must be a cryptographically random string of at least 43 characters.

**Rationale:** PKCE prevents authorization code interception by malicious apps that register the same custom URL scheme. Without PKCE, a malicious app can steal the authorization code from the redirect.

#### MASVS-AUTH-1.3 - Support Passkeys as Primary Authentication

The app SHOULD support passkeys via `ASAuthorizationPlatformPublicKeyCredentialProvider` (iOS 16+). Passkeys sync via iCloud Keychain. On iOS 18+, the app SHOULD use automatic passkey upgrade prompts for existing password accounts.

**Rationale:** Passkeys eliminate password reuse, phishing, and credential stuffing. Private keys stay on device (or synced within the user's iCloud Keychain). Public keys register with the server. No shared secret crosses the network.

**iOS Refs:** `ASAuthorizationPlatformPublicKeyCredentialProvider`, `ASAuthorizationController`, `AutoFill` credential provider

#### MASVS-AUTH-1.4 - Store Tokens in Keychain

OAuth tokens, session tokens, and JWTs MUST be stored in the Keychain with `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`. Tokens MUST NOT be stored in UserDefaults, unencrypted files, or bundled resources.

**Rationale:** Stolen tokens allow session hijacking. Keychain storage with ThisDeviceOnly prevents sync to other devices and exposure in backups.

#### MASVS-AUTH-1.5 - Implement Token Refresh and Rotation

The app MUST use short-lived access tokens (5-15 minutes) with refresh token rotation (one-time use). Refresh tokens MUST be stored with stronger protection than access tokens, specifically in a biometric-bound Keychain item using `SecAccessControl` with `.biometryCurrentSet`.

**Rationale:** Short-lived tokens limit the compromise window. Refresh token rotation detects token theft because reuse of a rotated token signals compromise to the server.

#### MASVS-AUTH-1.6 - Clear All Authentication State on Logout

On logout, the app MUST: revoke the refresh token server-side, delete all Keychain-stored tokens, clear `ASWebAuthenticationSession` cookies if used, clear `WKWebView` data stores, and navigate to the login screen.

**Rationale:** Incomplete logout leaves residual authentication state for subsequent device users or malware to exploit.

#### MASVS-AUTH-1.7 - Consider DPoP for High-Security Apps

For high-security apps, access tokens SHOULD be sender-constrained using DPoP (RFC 9449). DPoP proof keys SHOULD be Secure Enclave-backed via `SecureEnclave.P256`.

**Rationale:** Mobile devices have a unique advantage for DPoP because proof keys can be hardware-bound in the Secure Enclave, making stolen tokens unusable without the private key.

---

## MASVS-AUTH-2

### Control

The app performs local authentication securely according to platform best practices.

### Description

iOS apps authenticate users locally via Face ID or Touch ID using LAContext and Keychain integration. Local authentication must chain the biometric result to a Keychain cryptographic operation, not just check a boolean. A standalone `evaluatePolicy` boolean check is trivially bypassable with runtime instrumentation.

### iOS Sub-Requirements

#### MASVS-AUTH-2.1 - Chain Biometric to Keychain Item Access

Biometric authentication MUST gate a Keychain item read or decrypt operation via `SecAccessControl`, not just an `LAContext.evaluatePolicy` boolean check. The Keychain query MUST use `kSecUseAuthenticationContext` with an LAContext whose biometric result is verified by the Secure Enclave.

**Rationale:** An LAContext boolean can be hooked and forced to return true via Frida or other instrumentation tools. Chaining biometric authentication to a Keychain operation means the Secure Enclave must verify the biometric before releasing the key material.

**iOS Refs:** `SecAccessControlCreateWithFlags(.biometryCurrentSet)`, Keychain query with `kSecUseAuthenticationContext`

#### MASVS-AUTH-2.2 - Use .biometryCurrentSet for Biometric Invalidation

Keychain items gated by biometrics MUST use `.biometryCurrentSet`, not `.biometryAny` or `.userPresence`.

**Rationale:** `.biometryCurrentSet` invalidates the Keychain item if biometric enrollment changes (new fingerprint or face added). `.biometryAny` allows a newly enrolled biometric to access existing protected items, which means an attacker who adds their own biometric gains access.

#### MASVS-AUTH-2.3 - Handle LAError Cases

The app MUST handle all `LAError` cases: `.biometryNotAvailable`, `.biometryNotEnrolled`, `.biometryLockout`, `.passcodeNotSet`, `.userCancel`, `.userFallback`, and `.appCancel`. Each error MUST result in a defined behavior (fallback, message, or session termination).

**Rationale:** Unhandled error cases lead to crashes or security bypasses where the app proceeds as if authentication succeeded.

#### MASVS-AUTH-2.4 - Respect Platform Lockout

The app MUST rely on the platform's biometric lockout behavior (automatic after 5 failed attempts). The app MUST NOT implement custom retry logic that could bypass or override platform lockout.

**Rationale:** iOS enforces biometric lockout at the Secure Enclave level. Custom retry logic that resets counters or re-prompts undermines this hardware-backed protection.

#### MASVS-AUTH-2.5 - Provide Passcode Fallback Where Appropriate

For non-critical operations, the app SHOULD allow device passcode as a fallback via `.deviceOwnerAuthentication` (biometric or passcode). For security-critical operations (payment authorization, key access), the app MUST require `.deviceOwnerAuthenticationWithBiometrics` only.

**Rationale:** Passcode fallback improves usability for general access. Critical operations benefit from the stronger biometric factor, which is harder to shoulder-surf than a numeric passcode.

#### MASVS-AUTH-2.6 - Display Clear Authentication Purpose Strings

The app MUST provide meaningful `localizedReason` strings for LAContext prompts that explain why authentication is needed (e.g., "Authenticate to view your account balance"). Generic strings like "Please authenticate" MUST NOT be used.

**Rationale:** Generic prompts confuse users and train them to approve biometric requests without understanding the action being authorized.

---

## MASVS-AUTH-3

### Control

The app secures sensitive operations with additional authentication.

### Description

Certain operations require re-authentication beyond the initial login: changing security settings, authorizing payments, exporting data, or performing irreversible actions. On iOS, step-up authentication uses LAContext with Keychain-bound keys or server-side re-authentication. Client-side gates alone are insufficient; the server must independently verify that step-up authentication occurred.

### iOS Sub-Requirements

#### MASVS-AUTH-3.1 - Gate Sensitive Operations with Step-Up Authentication

The app MUST require re-authentication (biometric, passcode, or server-side) before the following operations: password changes, payment authorization, data export or deletion, new device authorization, and viewing highly sensitive PII.

**Rationale:** Initial authentication may have occurred minutes or hours ago. Re-authentication at the point of action confirms that the current user authorized the specific operation.

#### MASVS-AUTH-3.2 - Bind Step-Up Auth to Keychain Key Access

Step-up re-authentication MUST chain to a Keychain item with a per-use biometric requirement. The cryptographic operation (e.g., signing a request) only completes after biometric verification succeeds. The resulting signature is sent to the server for verification.

**Rationale:** A per-use biometric requirement forces the Secure Enclave to verify the biometric for each operation, preventing replay of previously authorized actions.

#### MASVS-AUTH-3.3 - Server Must Verify Cryptographic Proof

The server MUST independently verify that step-up authentication occurred. Client-side gates alone are insufficient. The server MUST verify a cryptographic proof such as a signed challenge via App Attest assertion or a Secure Enclave signature.

**Rationale:** Client-side checks can be bypassed with runtime instrumentation. Only server-side verification of a cryptographic proof provides reliable confirmation of step-up authentication.

#### MASVS-AUTH-3.4 - Support Multi-Factor Authentication

High-risk apps MUST support MFA combining at least two of: knowledge (password, PIN), possession (device with passkey, TOTP), and inherence (biometric). The factors must be independent so that compromise of one does not grant access.

**Rationale:** Single-factor authentication is vulnerable to targeted attacks. MFA forces an attacker to compromise multiple independent factors.

#### MASVS-AUTH-3.5 - Rate-Limit Step-Up Failures

The app and server MUST enforce rate limiting and lockout for step-up authentication failures. The session MUST be terminated after a defined failure threshold.

**Rationale:** Without rate limiting, an attacker with device access can brute-force step-up authentication prompts.

---

## Training-Aligned Requirements

| ID | Requirement | Testable Verification |
|---|---|---|
| AUTH-IOS-1.1 | OAuth flows MUST use `ASWebAuthenticationSession`, not `WKWebView`. | Verify OAuth launches system browser, not an in-app web view. |
| AUTH-IOS-1.2 | All OAuth flows MUST use PKCE with S256. | Capture the OAuth authorization request and verify the `code_challenge` and `code_challenge_method=S256` parameters are present. |
| AUTH-IOS-1.3 | The app SHOULD support passkeys via AuthenticationServices. | Verify passkey registration and authentication flows complete successfully. |
| AUTH-IOS-1.4 | Tokens MUST be stored in Keychain with `ThisDeviceOnly`. | Inspect Keychain items and verify accessibility class is `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`. |
| AUTH-IOS-1.5 | Logout MUST revoke server tokens and clear all local state. | Verify Keychain contains no authentication tokens after logout completes. |
| AUTH-IOS-2.1 | Biometric auth MUST gate a Keychain operation, not just an LAContext boolean. | Hook `LAContext.evaluatePolicy` to return true; verify the protected Keychain item is still inaccessible without genuine biometric verification. |
| AUTH-IOS-2.2 | Keychain items MUST use `.biometryCurrentSet`. | Add a new biometric enrollment and verify the previously created Keychain item is invalidated. |
| AUTH-IOS-3.1 | Sensitive operations MUST require re-authentication. | Attempt a sensitive operation (e.g., payment, data export) and verify a biometric or password prompt appears before the operation proceeds. |
