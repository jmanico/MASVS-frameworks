# MASVS-NETWORK: Network Communication

## Overview

App Transport Security (ATS) enforces TLS 1.2+ with forward secrecy for all connections by default since iOS 9. Combined with URLSession's certificate validation, iOS apps have strong networking foundations. Make sure apps use these correctly and do not weaken them with ATS exceptions or trust evaluation overrides. Future TLS behavior changes in Apple platforms should be handled as version-specific guidance.

## ATS Defaults

| Requirement | Default |
|---|---|
| Protocol | TLS 1.2 minimum |
| Cipher suites | Forward secrecy required (ECDHE) |
| Certificate | Valid chain to trusted root CA |
| Key size | RSA 2048+ or ECC 256+ |

## iOS Version Networking Behavior

| iOS Version | Behavior |
|---|---|
| 9+ | ATS enabled by default |
| 12.2+ | TLS 1.3 supported |
| 16+ | Paste permission prompt (affects clipboard-based data exchange) |
| Future releases | Additional TLS behavior may change; verify against current Apple documentation before making release-specific claims |

## OWASP Mobile Top 10 2024 Mapping

- **M5 - Insecure Communication:** Directly addressed by both controls
- **M8 - Security Misconfiguration:** Addressed by MASVS-NETWORK-1.1, MASVS-NETWORK-1.3, MASVS-NETWORK-1.4

---

## MASVS-NETWORK-1

### Control

The app secures all network traffic according to current best practices.

### Description

ATS provides declarative TLS enforcement. URLSession handles certificate validation. Make sure the app does not weaken these defaults.

### iOS Sub-Requirements

#### MASVS-NETWORK-1.1 - Do Not Disable ATS Globally

The app does not set `NSAllowsArbitraryLoads = YES` in Info.plist. Per-domain exceptions are used only with documented justification.

**Rationale:** Global ATS disable allows cleartext and weak TLS for all connections. App Store review may reject apps with blanket ATS disables.

#### MASVS-NETWORK-1.2 - TLS 1.2 Minimum, Prefer TLS 1.3

ATS defaults enforce TLS 1.2+. TLS 1.3 is supported since iOS 12.2 and preferred for improved security and performance.

**Rationale:** TLS 1.2 is the minimum acceptable version. TLS 1.3 removes legacy cipher suites and reduces handshake latency.

#### MASVS-NETWORK-1.3 - Do Not Override Trust Evaluation to Accept Invalid Certificates

The URLSessionDelegate does not call `completionHandler(.useCredential, ...)` for server trust challenges without proper validation. Never implement trust evaluation that accepts all certificates or ignores hostname mismatches.

**Rationale:** Trust-all implementations completely negate TLS security. A single permissive delegate method renders the entire TLS stack useless.

#### MASVS-NETWORK-1.4 - Do Not Include Debug ATS Exceptions in Release Builds

Per-domain ATS exceptions intended for development (`NSExceptionAllowsInsecureHTTPLoads`) are removed from release Info.plist.

**Rationale:** Stale debug exceptions create production vulnerabilities. Audit Info.plist ATS settings before every release.

#### MASVS-NETWORK-1.5 - Use SecTrustEvaluateWithError for Custom Validation

If the app performs custom certificate handling, use `SecTrustEvaluateWithError` (synchronous, iOS 12+) with the platform's default trust anchors. Do not implement custom validation that weakens the default chain.

**Rationale:** Custom trust evaluation is error-prone. Use the platform default whenever possible.

#### MASVS-NETWORK-1.6 - NSAllowsLocalNetworking Only for Local Services

`NSAllowsLocalNetworking` is only used for Bonjour/mDNS local discovery, not as a workaround for cleartext connections to remote servers.

**Rationale:** This exception is meant for local network protocols, not as a backdoor for HTTP.

#### MASVS-NETWORK-1.7 - Secure Non-HTTP Protocols

WebSocket, gRPC, MQTT, and other non-HTTP connections also use TLS. The same TLS requirements apply to all transport protocols.

**Rationale:** Network security requirements apply to all protocols equally. A cleartext WebSocket is just as dangerous as cleartext HTTP.

#### MASVS-NETWORK-1.8 - Fail Closed on TLS Failure

If a TLS connection fails, the app does not fall back to unencrypted HTTP. All network communication fails closed with no plaintext fallback.

**Rationale:** Fallback to cleartext defeats the purpose of TLS enforcement.

---

## MASVS-NETWORK-2

### Control

The app verifies server identity appropriately for remote endpoints under the developer's control.

### Description

Certificate Transparency provides assurance that certificates were publicly logged. Certificate pinning restricts trusted certificates to specific keys. Make sure the app verifies server identity using the platform trust model and adds pinning only where the risk model justifies it.

### iOS Sub-Requirements

#### MASVS-NETWORK-2.1 - Use Platform Trust Validation and Certificate Transparency

iOS platform trust validation and certificate transparency checks provide the default server-identity baseline for most apps. Where the product relies on the standard Apple trust model, no additional app-side identity mechanism is required beyond correct ATS and trust-evaluation handling.

**Rationale:** CT provides public accountability for certificate issuance without the operational risks of pinning.

#### MASVS-NETWORK-2.2 - Use Certificate Pinning Only Where Justified

Certificate pinning SHOULD be limited to high-security or high-assurance use cases with mature rotation procedures. If used, pin to SPKI (not the full certificate), include backup pins, and maintain documented rotation procedures.

**Rationale:** Pinning without proper rotation procedures causes outages when certificates change.

#### MASVS-NETWORK-2.3 - Full Certificate Chain Validation

The app validates the full certificate chain to a trusted root CA. Self-signed certificates are not accepted in production builds.

**Rationale:** Self-signed certificates offer no assurance of server identity.

#### MASVS-NETWORK-2.4 - Audit ATS Exceptions Before Every Release

The release process includes an audit of Info.plist for ATS exceptions. Stale or overly broad exceptions are removed.

**Rationale:** ATS exceptions added during development are often forgotten in production.

#### MASVS-NETWORK-2.5 - Secure API Communication Patterns

All API requests use HTTPS. Tokens go in the Authorization header, never in URL query parameters. Sensitive responses include `Cache-Control: no-store`. Request signing (HMAC or DPoP) SHOULD be used for sensitive operations.

**Rationale:** Tokens in query parameters leak into server logs, proxy logs, and analytics.

---

## Training-Aligned Requirements

| ID | Topic | Requirement | Test |
|---|---|---|---|
| NETWORK-IOS-1.1 | ATS Configuration | The app MUST NOT use `NSAllowsArbitraryLoads`. | Inspect Info.plist for ATS settings. |
| NETWORK-IOS-1.2 | TLS Version | The app MUST use TLS 1.2+ (ATS default). SHOULD prefer TLS 1.3. | Capture TLS handshake. |
| NETWORK-IOS-1.3 | Trust Evaluation | The app MUST NOT override trust evaluation to accept all certificates. | Static analysis for URLSessionDelegate trust handling. |
| NETWORK-IOS-1.4 | Fail Closed | The app MUST NOT fall back to HTTP on TLS failure. | Simulate cert failure, verify no cleartext retry. |
| NETWORK-IOS-2.1 | Certificate Chain | The app MUST validate the full chain. Self-signed certs MUST be rejected in production. | Attempt connection with self-signed cert. |
| NETWORK-IOS-2.2 | API Security | Tokens MUST be in Authorization header, not URL params. | Inspect API traffic. |
