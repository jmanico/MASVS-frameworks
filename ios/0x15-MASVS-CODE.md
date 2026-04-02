# MASVS-CODE: Code Quality and Supply Chain

## Overview

iOS apps are distributed as signed IPA bundles through the App Store. Every binary must be code-signed, verified at page-level execution. Dependencies come via Swift Package Manager, CocoaPods, or Carthage. The distribution model creates attack surface from outdated dependencies, malicious supply chain injections, and insufficient input validation.

## iOS Build and Distribution Security

| Concern | iOS Mechanism |
|---|---|
| Code signing | Mandatory Xcode code signing with provisioning profiles |
| Distribution | App Store (reviewed), TestFlight (beta), Enterprise, Ad Hoc |
| Dependency management | Swift Package Manager, CocoaPods, Carthage |
| Dependency locking | Package.resolved (SPM), Podfile.lock (CocoaPods) |
| SBOM generation | CycloneDX, Syft |
| App thinning | Bitcode (deprecated), app slicing |

## Input Validation Attack Surface

| Input Source | Attack | Mitigation |
|---|---|---|
| URL schemes / Universal Links | Parameter injection | Validate all parameters as untrusted |
| UIPasteboard | Malicious content injection | Validate pasted content |
| Handoff / NSUserActivity | Activity data injection | Validate continuation data |
| WKWebView content | XSS, JS bridge abuse | Sanitize, use content rules |
| NFC / QR codes | Malicious URLs | Display URL, validate against allowlist |
| Siri Intents | Intent parameter manipulation | Validate all intent parameters |
| Files via share sheet | Malformed files | Validate file format and content |

## Controls Summary

| Control | Title |
|---|---|
| MASVS-CODE-1 | The app requires an up-to-date platform version |
| MASVS-CODE-2 | The app has a mechanism for enforcing app updates |
| MASVS-CODE-3 | The app only uses software components without known vulnerabilities |
| MASVS-CODE-4 | The app validates and sanitizes all untrusted inputs |

## OWASP Mobile Top 10 2024 Mapping

- **M2 - Inadequate Supply Chain Security:** Directly addressed by MASVS-CODE-3
- **M4 - Insufficient Input/Output Validation:** Directly addressed by MASVS-CODE-4
- **M7 - Insufficient Binary Protections:** Partially addressed (see MASVS-RESILIENCE for full coverage)

---

## MASVS-CODE-1

### Control

The app requires an up-to-date platform version.

### Description

Each iOS release includes security patches, privacy improvements, and new security features. Supporting older iOS versions keeps users exposed to known, patched vulnerabilities and prevents the app from using modern security mechanisms. The app targets and requires a sufficiently recent iOS version to benefit from current platform security protections.

### iOS Sub-Requirements

#### MASVS-CODE-1.1 - Set Minimum Deployment Target to Supported iOS Version

The minimum deployment target is iOS 16 or higher. iOS 16 provides passkey support, paste permission prompts, Lockdown Mode, and Safety Check. These features are unavailable on earlier versions and represent significant security improvements for end users.

**iOS References:**
- Minimum Deployment Target in Xcode project settings
- `LSMinimumSystemVersion` in `Info.plist`

#### MASVS-CODE-1.2 - Build Against Latest Stable SDK

The app builds with the latest stable Xcode and iOS SDK to adopt the latest security behaviors and APIs. Each Xcode release activates new security defaults and deprecates insecure patterns.

**Rationale:** Building against an older SDK delays adoption of security behavior changes that Apple activates per-SDK-version.

#### MASVS-CODE-1.3 - Prompt Users on Outdated iOS Versions

If running on an iOS version that no longer receives security patches, the app informs the user of the security risk. The app may restrict sensitive functionality on unsupported versions.

**Rationale:** Even with a current app, an outdated OS with unpatched kernel and framework vulnerabilities puts user data at risk.

#### MASVS-CODE-1.4 - Use Availability Checks for Security Features

The app uses `#available` and `@available` checks to adopt stronger security features on newer iOS versions while maintaining compatibility with the minimum deployment target.

**Rationale:** Progressive security enhancement gives users on newer devices the strongest available protections while older devices still function.

---

## MASVS-CODE-2

### Control

The app has a mechanism for enforcing app updates.

### Description

When critical security vulnerabilities are discovered in a released iOS app, users must update promptly. iOS has no direct equivalent of Android's Play In-App Updates API. Server-side version enforcement is the primary mechanism for prompting or requiring updates. The app checks a backend endpoint for the minimum required version and directs users to the App Store when an update is needed.

### iOS Sub-Requirements

#### MASVS-CODE-2.1 - Implement Server-Side Minimum Version Check

The app checks a server-side endpoint for the minimum required version on launch. If the running app version is below the minimum, the app blocks sensitive functionality and directs the user to the App Store.

**Rationale:** No iOS API exists for in-app update prompts. Server-side enforcement works universally and gives the security team the ability to force updates when a critical vulnerability is found.

**iOS References:**
- `Bundle.main.infoDictionary["CFBundleShortVersionString"]` for current app version
- `SKStoreProductViewController` or direct App Store URL for update redirection

#### MASVS-CODE-2.2 - Handle Update Failures Gracefully

If the user cannot or declines to update, the app restricts sensitive features and re-prompts with increasing urgency. The app does not crash or lose data when handling update flows.

#### MASVS-CODE-2.3 - Monitor Version Distribution

The app reports its version to the backend on each API call or session start, enabling the team to track adoption of security updates and identify how many users remain on vulnerable versions.

**Rationale:** Without version telemetry, the team has no visibility into how many users remain on vulnerable versions after a security update is released.

---

## MASVS-CODE-3

### Control

The app only uses software components without known vulnerabilities.

### Description

iOS apps include third-party libraries via Swift Package Manager, CocoaPods, or Carthage. Each dependency is potential attack surface. Known vulnerabilities in these dependencies can be exploited by attackers. Supply chain attacks through SDKs are a top threat: SpinOK affected 400M+ installs, and the Shai-Hulud npm worm targeted React Native apps. The app maintains awareness of its dependency tree and addresses known vulnerabilities.

### iOS Sub-Requirements

#### MASVS-CODE-3.1 - Scan Dependencies for Known Vulnerabilities

The build pipeline includes automated dependency vulnerability scanning using tools such as Snyk, Dependabot, or OWASP Dependency-Check. Scans run on every pull request and release build.

**Rationale:** New CVEs are published regularly for popular iOS libraries. Automated scanning catches known vulnerabilities before they ship to users.

#### MASVS-CODE-3.2 - Maintain a Software Bill of Materials (SBOM)

The app maintains an SBOM listing all direct and transitive dependencies, their versions, and licenses. The SBOM is generated as part of the build process and stored alongside release artifacts.

**Rationale:** An SBOM enables rapid impact assessment when new vulnerabilities are disclosed. Regulatory requirements (EU Cyber Resilience Act, US Executive Order 14028) increasingly mandate SBOMs.

**iOS References:**
- CycloneDX or Syft for SBOM generation
- `swift package show-dependencies` for SPM dependency tree

#### MASVS-CODE-3.3 - Pin Dependency Versions

`Package.resolved` (SPM) and `Podfile.lock` (CocoaPods) are committed to source control. No dynamic version ranges in production builds. All dependencies use exact version specifications.

**Rationale:** Dynamic versions can resolve to different (potentially vulnerable or malicious) versions between builds. Version pinning prevents supply chain attacks via version hijacking and produces reproducible builds.

#### MASVS-CODE-3.4 - Verify Dependency Integrity

Swift Package Manager verifies package checksums automatically. For CocoaPods, podspec checksums are verified. Any checksum mismatch fails the build.

**Rationale:** Without integrity verification, a compromised package registry or MITM attack on the build could substitute malicious artifacts with matching version numbers.

#### MASVS-CODE-3.5 - Minimize Third-Party SDK Attack Surface

Each third-party SDK is evaluated before integration for: what data it collects (for App Store privacy labels), what permissions it requires, its maintenance status and security track record, and whether a platform API can replace it.

**Rationale:** Every third-party SDK increases the attack surface, may collect data without the user's knowledge, and may introduce vulnerabilities. SDK supply chain attacks are an increasing concern (OWASP Mobile Top 10 2024 M2).

---

## MASVS-CODE-4

### Control

The app validates and sanitizes all untrusted inputs.

### Description

iOS apps receive input from many sources: URL schemes, Universal Links, pasteboard, Handoff, WKWebView, NFC, QR codes, Siri Intents, share sheets, and files from other apps. All of these inputs can be crafted by an attacker. The app treats all external input as untrusted and applies appropriate validation and sanitization to prevent injection attacks and logic bypasses.

### iOS Sub-Requirements

#### MASVS-CODE-4.1 - Validate URL Scheme and Universal Link Parameters

All parameters from URL schemes and Universal Links are validated against expected types, ranges, and patterns before use. Parameters are never passed directly to sensitive operations without validation.

**Rationale:** Any app or website can invoke URL schemes and Universal Links. Malicious parameters can trigger unintended navigation, data exfiltration, or logic bypasses.

**iOS References:**
- `application(_:open:options:)` for URL schemes
- `application(_:continue:restorationHandler:)` for Universal Links

#### MASVS-CODE-4.2 - Sanitize Data Before WebView Loading

Data loaded into WKWebView via `loadHTMLString(_:baseURL:)` or `evaluateJavaScript(_:completionHandler:)` is sanitized for HTML and JavaScript injection.

**Rationale:** Unsanitized data in WebView content enables XSS within the app's WebView context. If a JavaScript message handler bridge is present, XSS escalates to native code execution.

#### MASVS-CODE-4.3 - Validate Pasteboard Content

Content pasted from UIPasteboard is validated before use in sensitive operations. The app does not blindly trust pasteboard data for URLs, account numbers, or other structured input.

**Rationale:** Any app can write to the general pasteboard. Malicious apps can place crafted content to exploit paste-based workflows.

#### MASVS-CODE-4.4 - Handle Deserialization Safely

Objects decoded via NSCoding use NSSecureCoding with explicit class allowlists. Codable types validate decoded values after deserialization. The app never decodes arbitrary classes from untrusted data.

**Rationale:** Insecure deserialization can trigger arbitrary object instantiation and unexpected behavior.

**iOS References:**
- `NSSecureCoding` protocol
- `NSKeyedUnarchiver.requiresSecureCoding = true`
- `unarchivedObject(ofClass:from:)` with explicit class parameter

#### MASVS-CODE-4.5 - Validate QR Code and NFC Input

Decoded QR code and NFC data is treated as untrusted. URLs are displayed to the user before navigation. The app never auto-executes actions based on scanned data.

**Rationale:** QR codes can be placed in public locations by anyone. NFC tags can be written and repositioned by attackers. Auto-executing scanned URLs or commands is a direct path to phishing or exploitation.

#### MASVS-CODE-4.6 - Validate Siri Intent and Share Sheet Input

All parameters from SiriKit intents and share sheet input are validated against expected types and ranges before processing.

**Rationale:** Intent and share sheet parameters originate from external apps or system services and can contain unexpected or malicious values.

**iOS References:**
- `INIntent` subclass parameter validation
- `NSExtensionContext.inputItems` validation in share extensions

---

## Training-Aligned Requirements

iOS-specific requirements for MASVS-CODE-1 through MASVS-CODE-4.

### MASVS-CODE-1: Secure Build Infrastructure

#### CODE-IOS-1.1: Code Signing

The app MUST use Xcode Managed Signing or documented manual signing with provisioning profiles. Signing keys and certificates MUST NOT be stored in source control.

**Testable:** Verify signing configuration in Xcode project settings. Verify no signing keys or `.p12` files in repository.

#### CODE-IOS-1.2: CI/CD Security Pipeline

The build pipeline MUST include SAST scans, dependency vulnerability scanning, and SBOM generation. Signing MUST use Keychain or a secret vault, not plaintext credentials.

**Testable:** Review CI/CD configuration. Verify SAST, dependency scanning, and SBOM generation steps are present and passing.

#### CODE-IOS-1.3: Debug Build Separation

Release builds MUST NOT contain debug symbols, test endpoints, or verbose logging. `DEBUG` preprocessor flags MUST gate all debug-only code paths.

**Testable:** Inspect release IPA for debug symbols, test URLs, and verbose log statements.

### MASVS-CODE-2: Secure Third-Party Libraries

#### CODE-IOS-2.1: SDK Vetting

All third-party SDKs MUST be vetted before integration: review declared permissions, data collection behavior, network traffic, and transitive dependencies. An SBOM MUST exist and be current.

**Testable:** Verify SBOM exists and is current. Verify SDK vetting documentation exists for each integrated SDK.

#### CODE-IOS-2.2: Dependency Monitoring

Dependencies MUST be continuously monitored for known CVEs. Critical vulnerabilities MUST be updated within 72 hours. High-severity vulnerabilities MUST be updated within 7 days.

**Testable:** Verify dependency scanning runs in CI/CD. Review vulnerability response SLA compliance.

### MASVS-CODE-3: Input Validation

#### CODE-IOS-3.1: Untrusted Input Validation

All data from external sources (deep links, Universal Links, pasteboard, QR codes, NFC, Siri Intents, share sheet) MUST be validated and sanitized before processing.

**Testable:** Send malformed and malicious data via each input vector. Verify the app rejects or safely handles invalid input.

#### CODE-IOS-3.2: NSSecureCoding

All NSCoding deserialization MUST use NSSecureCoding with explicit class allowlists. `NSKeyedUnarchiver` MUST have `requiresSecureCoding` set to `true`.

**Testable:** Verify NSSecureCoding adoption across all archived object types. Verify no usage of `unarchiveObject(with:)` (deprecated, insecure).
