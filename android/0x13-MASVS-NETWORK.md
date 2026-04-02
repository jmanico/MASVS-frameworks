# MASVS-NETWORK: Network Communication

## Overview

Android provides a robust, declarative system for network security through the Network Security Configuration (`network_security_config.xml`). This system controls cleartext traffic policy, certificate trust anchors, certificate pinning, and debug overrides — all without writing code. Combined with the platform's default TLS support (TLS 1.2 minimum since API 20, TLS 1.3 since API 29), Android apps have strong foundations for secure networking. This category ensures apps leverage these mechanisms correctly and do not undermine them with insecure custom implementations.

## Network Security Configuration Reference

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2027-01-01">
            <pin digest="SHA-256">primaryPinHash=</pin>
            <pin digest="SHA-256">backupPinHash=</pin>
        </pin-set>
    </domain-config>
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

## Android Version-Specific Behaviors

| API Level | Behavior |
|---|---|
| 24 (Android 7) | User-installed CAs no longer trusted by default |
| 28 (Android 9) | Cleartext traffic blocked by default |
| 29 (Android 10) | TLS 1.3 support enabled by default |
| 35 (Android 15) | TLS 1.0 and 1.1 disabled for targeting apps |

## Controls Summary

| Control | Title |
|---|---|
| MASVS-NETWORK-1 | The app secures all network traffic according to the current best practices |
| MASVS-NETWORK-2 | The app performs identity pinning for all remote endpoints under the developer's control |

## OWASP Mobile Top 10 2024 Mapping

- **M5 — Insecure Communication:** Directly addressed by both controls
- **M8 — Security Misconfiguration:** Addressed by MASVS-NETWORK-1.1, 1.3, 1.4

---

## MASVS-NETWORK-1

### Control

The app secures all network traffic according to the current best practices.

### Description

Ensuring data confidentiality and integrity for all network communication is critical for any Android app. Android provides the Network Security Configuration system for declarative TLS policy, enforces cleartext traffic restrictions by default since API 28, and offers TLS 1.3 support. This control ensures the app correctly configures and maintains secure network connections for all traffic, using Android-specific mechanisms.

### Android Sub-Requirements

#### MASVS-NETWORK-1.1 — Disable Cleartext Traffic

The app blocks all cleartext (HTTP) traffic by configuring `cleartextTrafficPermitted="false"` in `network_security_config.xml` or verifying the default (false since `targetSdkVersion >= 28`). Per-domain exceptions are only used for documented, justified cases (e.g., localhost development).

**Android References:**
- XML example of `network_security_config.xml` with `cleartextTrafficPermitted="false"`
- `android:networkSecurityConfig="@xml/network_security_config"` in `<application>`
- `StrictMode.VmPolicy.Builder.detectCleartextNetwork()` for development-time detection

#### MASVS-NETWORK-1.2 — Enforce TLS 1.2 as Minimum, Prefer TLS 1.3

The app uses TLS 1.2 as the minimum protocol version. TLS 1.0 and 1.1 are not permitted (Android 15+ disables them by default for apps targeting API 35+). TLS 1.3 is preferred where supported.

**Rationale:** TLS 1.0 and 1.1 have known vulnerabilities (BEAST, POODLE, Lucky13). TLS 1.3 provides improved performance (1-RTT handshake) and security (no vulnerable cipher suites).

**Android References:**
- `SSLContext.getInstance("TLSv1.3")`
- `OkHttpClient.Builder().connectionSpecs(listOf(ConnectionSpec.MODERN_TLS))`

#### MASVS-NETWORK-1.3 — Do Not Override TrustManager or HostnameVerifier

The app does not implement custom `X509TrustManager` that accepts all certificates, custom `HostnameVerifier` that accepts all hostnames, or custom `SSLSocketFactory` that disables validation. These patterns are never present in production code, even behind feature flags or debug conditions.

**Rationale:** Trust-all and hostname-skip implementations completely negate TLS security, enabling any network attacker to intercept all traffic. These are the most common Android network security vulnerabilities.

#### MASVS-NETWORK-1.4 — Do Not Trust User-Installed CA Certificates in Production

The `network_security_config.xml` does not include `<certificates src="user" />` in the `<base-config>` trust anchors for release builds. Only system CAs are trusted.

**Rationale:** User-installed CA certificates enable MITM proxying of all app traffic by any party that can install a certificate on the device (MDM, employer, attacker with device access). Since API 24, user CAs are not trusted by default.

**Android References:**
- Default behavior since API 24 (`targetSdkVersion >= 24`): only system CAs trusted
- Debug-only exception via `<debug-overrides>` in Network Security Config

#### MASVS-NETWORK-1.5 — Validate Server Certificates Properly

The app relies on the platform's default certificate validation chain (system trust store, Network Security Config). If the app performs any custom certificate handling, it uses `CertPathValidator` with the platform's default `TrustAnchor` set and does not bypass validation for any certificate error.

#### MASVS-NETWORK-1.6 — Use Network Security Config for All TLS Policy

All TLS configuration (trust anchors, cleartext policy, certificate pinning, debug overrides) is managed through `res/xml/network_security_config.xml`, not through programmatic SSL/TLS configuration in code. This provides a declarative, auditable, centralized security policy.

**Rationale:** Declarative configuration in Network Security Config is easier to audit, less error-prone than programmatic configuration, and survives code changes that might accidentally weaken TLS settings.

#### MASVS-NETWORK-1.7 — Secure WebSocket and Non-HTTP Connections

If the app uses WebSocket, gRPC, MQTT, or other non-HTTP protocols, these connections also use TLS (WSS, gRPC with TLS, MQTTS). The same TLS requirements (minimum version, certificate validation, no cleartext) apply to all network protocols, not just HTTP.

**Rationale:** Network security requirements apply to all transport protocols. WebSocket over `ws://` or MQTT over plaintext are equally vulnerable to interception as HTTP.

#### MASVS-NETWORK-1.8 — Consider DNS Security

The app is aware that standard DNS queries are unencrypted and can be intercepted or spoofed. For high-security applications, the app uses DNS-over-HTTPS (DoH) via `OkHttp` or `android.net.DnsResolver`, or relies on the user's Private DNS configuration (DoT, available since Android 9).

**Rationale:** DNS queries reveal which servers the app communicates with and can be spoofed to redirect traffic to attacker-controlled servers, even when TLS is used (if certificate validation is then bypassed).

---

## MASVS-NETWORK-2

### Control

The app performs identity pinning for all remote endpoints under the developer's control.

### Description

Certificate pinning restricts which certificates or public keys are trusted for specific domains, rather than trusting the entire system CA store. On Android, this is best implemented declaratively via the Network Security Configuration. Pinning protects against compromised CAs, rogue certificates, and sophisticated MITM attacks. This control ensures pinning is implemented correctly with proper backup pins and expiration handling.

### Android Sub-Requirements

#### MASVS-NETWORK-2.1 — Implement Certificate Pinning via Network Security Config

The app configures certificate pinning in `network_security_config.xml` using `<pin-set>` for all remote endpoints under the developer's control. Pins are SPKI (SubjectPublicKeyInfo) SHA-256 hashes.

**Rationale:** Declarative pinning via Network Security Config is preferred over programmatic pinning (OkHttp `CertificatePinner`) because it is centralized, auditable, and managed independently of networking code.

#### MASVS-NETWORK-2.2 — Include Backup Pins for Key Rotation

Every `<pin-set>` includes at least one backup pin that corresponds to a key not currently in production. This ensures the app can continue to function during emergency key rotation without requiring an app update.

**Rationale:** If the production key is compromised or its certificate expires and no backup pin is configured, the app becomes non-functional until users update to a version with new pins. Backup pins provide a recovery path.

#### MASVS-NETWORK-2.3 — Set Pin Expiration Dates

Every `<pin-set>` includes an `expiration` attribute with a date that is far enough in the future to allow planned key rotation but not so far that stale pins persist indefinitely. After expiration, the platform falls back to standard certificate validation.

**Rationale:** Pinning without expiration creates a permanent bricking risk if all pinned keys are rotated and users have not updated the app. Expiration provides a safety valve.

#### MASVS-NETWORK-2.4 — Pin Against SPKI Hashes, Not Certificate Hashes

Pins are computed against the SubjectPublicKeyInfo (SPKI) of the certificate, not the entire certificate. This allows certificate renewal without changing the pin, as long as the same public key is reused.

**Rationale:** Certificates are reissued regularly (e.g., Let's Encrypt every 90 days). Pinning the full certificate would break the app on every reissue. SPKI pinning survives certificate renewal if the key pair is preserved.

#### MASVS-NETWORK-2.5 — Implement Pinning Failure Reporting

When a pin validation failure occurs, the app reports the failure to a server-side monitoring endpoint (with the domain, expected pins, and received certificate chain) for security incident detection. The app does not silently fall back to unpinned connections.

**Rationale:** Pin validation failures may indicate an active MITM attack. Reporting enables the security team to investigate and respond.

#### MASVS-NETWORK-2.6 — Consider Certificate Transparency

For additional protection, the app verifies that server certificates include Signed Certificate Timestamps (SCTs) from Certificate Transparency logs. This provides an additional layer of assurance that certificates were publicly logged and not secretly issued.

**Rationale:** Certificate Transparency detects misissued or rogue certificates by requiring public logging. Combined with pinning, CT provides defense-in-depth against CA compromise.

---

## Training-Aligned Requirements

Android-specific requirements for MASVS-NETWORK-1 and MASVS-NETWORK-2.

### MASVS-NETWORK-1: Secure Network Traffic

#### NETWORK-ANDROID-1.1: Network Security Configuration

The app MUST include a Network Security Configuration (NSC) file that: Sets `cleartextTrafficPermitted="false"` as the base configuration, Does NOT include `<debug-overrides>` with user-added CAs in release builds, Does NOT include per-domain cleartext exceptions unless absolutely necessary and documented.

**Testable:** Inspect `res/xml/network_security_config.xml`. Verify `cleartextTrafficPermitted="false"` in base-config. Verify no debug-overrides in release APK.

#### NETWORK-ANDROID-1.2: TLS 1.3 Preferred

The app MUST support TLS 1.2 as minimum and SHOULD prefer TLS 1.3 (available since Android 10/API 29). TLS 1.0 and 1.1 MUST NOT be used.

**Testable:** Capture TLS handshake with proxy. Verify negotiated TLS version is 1.2 or 1.3. No successful connections on TLS 1.0/1.1.

#### NETWORK-ANDROID-1.3: Fail-Closed on TLS Failure

If a TLS connection fails, the app MUST NOT fall back to unencrypted HTTP. All network communication MUST fail closed — no plaintext fallback under any circumstance.

**Testable:** Simulate TLS failure (invalid cert, handshake timeout). Verify app does not retry over HTTP. Verify appropriate error is displayed.

#### NETWORK-ANDROID-1.4: No Trust Manager Override

The app MUST NOT implement custom `TrustManager` or `HostnameVerifier` that bypasses certificate validation. `X509TrustManager.checkServerTrusted()` MUST NOT be implemented as a no-op.

**Testable:** Static analysis for custom `TrustManager`, `HostnameVerifier`, or `SSLSocketFactory` implementations. Verify none accept all certificates.

#### NETWORK-ANDROID-1.5: Secure API Communication

All API requests MUST use HTTPS. Authentication tokens MUST be sent in the `Authorization` header, never in URL query parameters. Sensitive API responses MUST include `Cache-Control: no-store` headers. Request signing (HMAC or DPoP) SHOULD be used for sensitive operations.

**Testable:** Inspect all API traffic. Verify HTTPS-only, header-based auth, no token leakage in URLs, appropriate cache headers.

#### NETWORK-ANDROID-1.6: Android 17 Cleartext Deprecation

For apps targeting Android 17 (API 37+), the `usesCleartextTraffic` manifest attribute is deprecated. The app MUST use Network Security Configuration exclusively for traffic policy.

**Testable:** Verify apps targeting SDK 37+ use NSC instead of manifest attribute for cleartext policy.

### MASVS-NETWORK-2: Certificate Transparency and Pinning

#### NETWORK-ANDROID-2.1: Certificate Transparency

On Android 17+, Certificate Transparency (CT) is enabled by default. For earlier Android versions, the app SHOULD verify CT compliance of server certificates where feasible.

**Rationale:** The industry has moved from certificate pinning to Certificate Transparency. Android 14 removed `pin-set` support from NSC. CT provides better security with lower operational risk than pinning (no risk of bricking the app on certificate rotation).

**Testable:** On Android 17+, verify CT enforcement is active (not disabled). On earlier versions, verify CT compliance checking if implemented.

#### NETWORK-ANDROID-2.2: Certificate Pinning (Legacy/High-Security Only)

Certificate pinning SHOULD NOT be used for most applications due to operational risks (certificate rotation failures, CA root expiry incidents like Let's Encrypt 2021). Pinning is acceptable ONLY for very high-security apps that have mature certificate rotation procedures and incident response plans. If pinning is used, it MUST: Pin to the public key (SPKI), not the full certificate; Include at least one backup pin; Implement a reporting mechanism for pin failures; Have a documented rotation procedure.

**Testable:** If pinning is present, verify SPKI pinning, backup pin, and failure reporting. If pinning is absent, verify CT compliance per NETWORK-ANDROID-2.1.

#### NETWORK-ANDROID-2.3: Full Certificate Chain Validation

The app MUST validate the full certificate chain to a trusted root CA. Self-signed certificates MUST NOT be accepted in production builds.

**Testable:** Attempt connection with self-signed certificate. Verify connection is rejected. Verify full chain validation via proxy inspection.
