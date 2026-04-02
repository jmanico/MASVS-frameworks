# MASVS-CODE: Code Quality and Supply Chain

## Overview

Android apps are distributed as APK/AAB bundles containing DEX bytecode, native libraries, resources, and third-party dependencies fetched via Gradle from Maven repositories. This distribution model creates attack surface at multiple levels - outdated platform dependencies, unpatched third-party libraries, malicious supply chain injections, and insufficient input validation. Make sure the app maintains code quality, manages its dependency supply chain securely, and validates all untrusted inputs.

## Android Build and Distribution Security

| Concern | Android Mechanism |
|---|---|
| Platform version | `minSdkVersion` / `targetSdkVersion` in `build.gradle` |
| Forced updates | Google Play In-App Updates API (`com.google.android.play:app-update`) |
| Dependency scanning | OWASP Dependency-Check, Snyk, GitHub Dependabot |
| Dependency integrity | `gradle/verification-metadata.xml` (SHA-256 + PGP verification) |
| Dependency locking | `dependenciesLocking { lockAllConfigurations() }` + `gradle.lockfile` |
| SBOM generation | CycloneDX Gradle plugin |
| App signing | APK Signature Scheme v2/v3/v4 via `apksigner` |
| Code shrinking | R8 (`minifyEnabled true`) |
| Resource shrinking | R8 (`shrinkResources true`) |

## Input Validation Attack Surface

| Input Source | Attack | Mitigation |
|---|---|---|
| Intent extras/data | Type confusion, deserialization | Type-safe `getParcelableExtra()` with class param (API 33+) |
| Content Provider queries | SQL injection | Parameterized queries with `selectionArgs` |
| `ContentProvider.openFile()` | Path traversal | Canonical path validation |
| ZIP archives | ZipSlip (path traversal) | Validate `ZipEntry` names before extraction |
| WebView content | XSS, JavaScript injection | HTML sanitization before `loadData()` |
| Deep links | Parameter injection | URI scheme/host/param validation |
| NFC/Bluetooth data | Malformed messages | Format and range validation |

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

Each Android release includes security patches, privacy improvements, and new security features. Supporting older Android versions keeps users exposed to known, patched vulnerabilities and prevents the app from using modern security mechanisms. Make sure the app targets and requires a sufficiently recent Android version to benefit from current platform security protections.

### Android Sub-Requirements

#### MASVS-CODE-1.1 - Set Appropriate minSdkVersion

The app's `minSdkVersion` is set to at least API 28 (Android 9) to ensure baseline security features including: Network Security Configuration with cleartext blocking by default, StrongBox Keystore support, BiometricPrompt API availability, Protected Confirmation support, Hardware security module attestation. For new apps, `minSdkVersion` of API 29 (Android 10) or higher is recommended to benefit from scoped storage, background location restrictions, and TLS 1.3 by default.

**Android References:**
- `minSdkVersion` in `build.gradle` / `build.gradle.kts`

#### MASVS-CODE-1.2 - Target the Latest Stable API Level

The app's `targetSdkVersion` is set to the latest stable API level (API 35 for Android 15, API 36 for Android 16) to opt in to the latest security behaviors: API 31+: `PendingIntent` mutability must be explicit; components with intent-filters must declare `android:exported`; API 33+: `RECEIVER_NOT_EXPORTED` flag for broadcast receiver registration; API 34+: Partial media access; stricter background restrictions; API 35+: TLS 1.0/1.1 disabled; safer intent matching; API 36+: Local network permission; Safer Intents v2; intent redirection hardening.

**Rationale:** Google Play requires targeting a recent API level. Each new target SDK activates security behavior changes that protect the app and its users.

#### MASVS-CODE-1.3 - Prompt Users on Outdated OS Versions

If the app detects it is running on an Android version that no longer receives security patches (typically versions more than 3 years old), it informs the user of the security risk. The app may choose to restrict sensitive functionality on unsupported OS versions.

**Rationale:** Even with a current app, an outdated OS with unpatched kernel and framework vulnerabilities puts user data at risk.

#### MASVS-CODE-1.4 - Use Android API Level Checks for Security Features

The app uses `Build.VERSION.SDK_INT` checks to use stronger security features on newer API levels while maintaining compatibility. Kotlin code example showing StrongBox fallback to TEE.

**Rationale:** Progressive security enhancement ensures users on newer devices get the strongest available protections while older devices still function.

---

## MASVS-CODE-2

### Control

The app has a mechanism for enforcing app updates.

### Description

When critical security vulnerabilities are discovered in a released Android app, users must be able to update promptly. Android provides the Google Play In-App Updates API for prompting users to update within the app. Make sure the app can force or strongly encourage updates when security-critical issues are discovered.

### Android Sub-Requirements

#### MASVS-CODE-2.1 - Implement In-App Update Checks

The app uses the Google Play In-App Updates API (`com.google.android.play:app-update`) to check for available updates and prompt the user. For critical security updates, the app uses the immediate update flow (full-screen, blocking) rather than the flexible (background download) flow.

**Android References:**
- `com.google.android.play:app-update` and `app-update-ktx` Gradle dependencies
- `AppUpdateManager.appUpdateInfo`
- `AppUpdateType.IMMEDIATE`
- `AppUpdateType.FLEXIBLE`

#### MASVS-CODE-2.2 - Implement Minimum Version Enforcement

The app checks a server-side minimum version endpoint on launch. If the running app version is below the minimum, the app blocks usage and directs the user to update.

**Rationale:** The In-App Updates API only works with Google Play. Server-side version enforcement works regardless of distribution channel and gives the security team the ability to force updates when a critical vulnerability is found.

#### MASVS-CODE-2.3 - Handle Update Failures Gracefully

If the user declines or fails to update when a critical update is available, the app either: Restricts access to sensitive functionality while running the outdated version; Re-prompts on next launch with increasing urgency; Blocks usage entirely if the vulnerability is critical enough. The app does not crash, lose data, or degrade unexpectedly when handling update flows.

#### MASVS-CODE-2.4 - Monitor Update Adoption

The app reports its version to the backend on each API call or session start, enabling the security team to monitor the distribution of app versions in the wild and track the adoption rate of critical security updates.

**Rationale:** Without version telemetry, the team has no visibility into how many users remain on vulnerable versions after a security update is released.

---

## MASVS-CODE-3

### Control

The app only uses software components without known vulnerabilities.

### Description

Android apps typically include dozens of third-party libraries via Gradle dependencies - networking (OkHttp, Retrofit), image loading (Glide, Coil), analytics, advertising, and more. Each dependency is potential attack surface. Known vulnerabilities in these dependencies can be exploited by attackers. Make sure the app maintains awareness of its dependency tree and addresses known vulnerabilities.

### Android Sub-Requirements

#### MASVS-CODE-3.1 - Scan Dependencies for Known Vulnerabilities

The build pipeline includes automated dependency vulnerability scanning using tools such as: OWASP Dependency-Check (`org.owasp:dependency-check-gradle`), Snyk, GitHub Dependabot, Google's `play-services-safetynet` / Play Integrity for runtime checks (limited). Scans run on every build or at minimum on every pull request and release build.

**Rationale:** New CVEs are published regularly for popular Android libraries. Automated scanning catches known vulnerabilities before they ship to users.

#### MASVS-CODE-3.2 - Maintain a Software Bill of Materials (SBOM)

The app maintains an SBOM listing all direct and transitive dependencies, their versions, and licenses. The SBOM is generated as part of the build process and stored alongside release artifacts.

**Rationale:** An SBOM enables rapid impact assessment when new vulnerabilities are disclosed. Regulatory requirements (EU Cyber Resilience Act, US Executive Order 14028) increasingly mandate SBOMs.

**Android References:**
- `./gradlew dependencies`
- CycloneDX Gradle plugin for SBOM generation in CycloneDX format

#### MASVS-CODE-3.3 - Pin Dependency Versions

All dependencies in `build.gradle` use exact version numbers (e.g., `2.9.0`), not dynamic versions (`2.+`, `latest.release`). Gradle dependency locking (`dependencyLocking`) is enabled to produce reproducible builds.

**Rationale:** Dynamic versions can resolve to different (potentially vulnerable or malicious) versions between builds. Version pinning ensures reproducibility and prevents supply chain attacks via version hijacking.

**Android References:**
- Gradle dependency locking: `dependenciesLocking { lockAllConfigurations() }`
- `gradle.lockfile`

#### MASVS-CODE-3.4 - Verify Dependency Integrity

Gradle dependency verification (`gradle/verification-metadata.xml`) is configured to verify checksums (SHA-256) and/or PGP signatures of all downloaded dependencies.

**Rationale:** Without integrity verification, a compromised Maven repository or MITM attack on the build could substitute malicious artifacts with matching version numbers.

**Android References:**
- `gradle/verification-metadata.xml`
- `./gradlew --write-verification-metadata sha256,pgp`

#### MASVS-CODE-3.5 - Minimize Third-Party SDK Attack Surface

The app includes only essential third-party SDKs. Each SDK is evaluated for: What data it collects (for Google Play Data Safety Section compliance), What permissions it requires, Whether it includes native code (`.so` libraries), Its maintenance status and security track record, Whether it can be replaced with a platform API or smaller, audited alternative.

**Rationale:** Every third-party SDK increases the attack surface, may collect data without the user's knowledge, and may introduce vulnerabilities. SDK supply chain attacks are an increasing concern (OWASP Mobile Top 10 2024 M2).

---

## MASVS-CODE-4

### Control

The app validates and sanitizes all untrusted inputs.

### Description

Android apps receive input from many sources: user interface fields, intents from other apps, deep links, content providers, the network, local files, clipboard, and NFC. All of these inputs can be crafted by an attacker. Make sure the app treats all external input as untrusted and applies appropriate validation and sanitization to prevent injection attacks and logic bypasses on Android.

### Android Sub-Requirements

#### MASVS-CODE-4.1 - Validate Intent Extras and Data URIs

All data extracted from incoming intents is validated against expected types, ranges, and patterns before use. Specific risks: `getParcelableExtra()` can trigger deserialization of attacker-controlled classes; `getData()` URIs may contain unexpected schemes; Missing extras should be handled gracefully.

**Android References:**
- Use `IntentCompat.getParcelableExtra()` with explicit class parameter (API 33+)
- Validate URI schemes against an allowlist

#### MASVS-CODE-4.2 - Prevent SQL Injection in Content Providers

Content Provider `query()`, `update()`, `delete()` operations use parameterized queries with `selectionArgs`. Java code examples showing secure vs insecure patterns.

#### MASVS-CODE-4.3 - Prevent Path Traversal in File Operations

The app validates file paths received from intents, content URIs, or user input. When implementing `ContentProvider.openFile()`, the resolved path is validated to be within the expected base directory. Java code example with canonical path validation.

#### MASVS-CODE-4.4 - Prevent Zip Path Traversal (ZipSlip)

When extracting ZIP archives, the app validates that each `ZipEntry` name does not contain path traversal sequences (`../`).

**Rationale:** Zip path traversal (ZipSlip) allows an attacker to overwrite arbitrary files on the filesystem by crafting ZIP entries with `../../` in their names.

#### MASVS-CODE-4.5 - Sanitize Data Before WebView Loading

Data loaded into WebViews via `loadUrl()`, `loadData()`, `loadDataWithBaseURL()`, or `evaluateJavascript()` is sanitized for HTML/JavaScript injection.

**Rationale:** Unsanitized data in WebView content enables XSS within the app's WebView context. If a JavaScript interface bridge is present, XSS escalates to native code execution.

#### MASVS-CODE-4.6 - Validate Deep Link Parameters

Parameters received via deep links are validated against expected patterns, types, and ranges.

**Rationale:** Deep links are externally accessible - any app or website can invoke them.

#### MASVS-CODE-4.7 - Handle Deserialization Safely

The app avoids Java serialization (`Serializable`) for untrusted data. If `Parcelable` objects are received from untrusted sources, the app uses explicit class parameters for `getParcelableExtra()` (API 33+).

**Rationale:** Java deserialization can trigger arbitrary code execution via gadget chains.

#### MASVS-CODE-4.8 - Validate NFC and Bluetooth Input

If the app processes NFC (NDEF messages) or Bluetooth data, all received data is validated.

**Rationale:** NFC tags can be written by anyone and placed in public locations. Bluetooth data can be spoofed.

---

## Training-Aligned Requirements

Android-specific requirements for MASVS-CODE-1 through MASVS-CODE-4.

### MASVS-CODE-1: Secure Build Infrastructure

#### CODE-ANDROID-1.1: Play App Signing

The app MUST use Play App Signing (Google-managed HSM) for production signing. Signing keys MUST NOT be stored on developer workstations or in source control.

**Testable:** Verify Play App Signing enrollment in Google Play Console. Verify no signing keys in repository.

#### CODE-ANDROID-1.2: APK Signature Scheme v3+

The app MUST use APK Signature Scheme v3 or v4 for signing integrity. Legacy v1 (JAR signing) MUST NOT be the sole signing mechanism.

**Testable:** Run `apksigner verify --print-certs` on the APK. Verify v3 or v4 scheme is present.

#### CODE-ANDROID-1.3: CI/CD Security Pipeline

The build pipeline MUST include: SAST scans, Dependency vulnerability scanning, SBOM generation, Signing via HSM or secret vault (not local keystore files).

**Testable:** Review CI/CD configuration. Verify SAST, dependency scanning, and SBOM generation steps are present and passing.

#### CODE-ANDROID-1.4: Reproducible Builds

The app SHOULD support reproducible builds.

**Testable:** Build from the same source twice. Compare output artifacts for identical hashes.

#### CODE-ANDROID-1.5: Debug Build Separation

Release builds MUST have `android:debuggable="false"`. Debug-specific code, logging, and test endpoints MUST be removed or disabled in release builds.

**Testable:** Inspect release APK manifest for `debuggable` flag. Verify no debug endpoints are reachable.

### MASVS-CODE-2: Secure Third-Party Libraries

#### CODE-ANDROID-2.1: SDK Supply Chain Vetting

All third-party SDKs MUST be vetted before integration: Review declared permissions and network behavior, Monitor network traffic with mitmproxy or equivalent, Generate and review SBOM for transitive dependencies, Document the decision to keep, replace, or remove each SDK.

**Rationale:** Supply chain attacks through SDKs are a top mobile threat. SpinOK SDK affected 101 apps with 400M+ installs.

**Testable:** Verify SBOM exists and is current. Verify SDK vetting documentation exists.

#### CODE-ANDROID-2.2: SDK Permission Auditing

Third-party SDKs MUST NOT request permissions beyond what is necessary for their functionality. SDK permissions MUST be audited and documented.

**Testable:** Compare SDK-declared permissions against documented functionality.

#### CODE-ANDROID-2.3: SDK Consent Enforcement

Third-party SDKs MUST respect user consent signals. SDKs MUST NOT collect data before consent is confirmed.

**Testable:** Monitor SDK network traffic before and after user consent.

#### CODE-ANDROID-2.4: Dependency Vulnerability Monitoring

Dependencies MUST be continuously monitored for known CVEs. Vulnerable dependencies MUST be updated within a defined SLA.

**Testable:** Verify dependency scanning runs in CI/CD.

### MASVS-CODE-3: Input Validation

#### CODE-ANDROID-3.1: Untrusted Input Validation

All data from external sources (deep links, Intents, QR codes, NFC, clipboard, IPC) MUST be validated and sanitized before processing.

**Testable:** Send malformed/malicious data via each input vector.

#### CODE-ANDROID-3.2: WebView Input Sanitization

Data passed to WebViews MUST be sanitized to prevent XSS and injection attacks.

**Testable:** Inject JavaScript payloads via WebView input vectors.

### MASVS-CODE-4: Security Update Readiness

#### CODE-ANDROID-4.1: Minimum SDK Version

The app MUST set `minSdkVersion` to a supported Android version (API 29 or higher).

**Testable:** Inspect `build.gradle` for `minSdkVersion`.

#### CODE-ANDROID-4.2: Target SDK Currency

The app MUST target a recent SDK version as required by Google Play policy.

**Testable:** Inspect `build.gradle` for `targetSdkVersion`.

#### CODE-ANDROID-4.3: In-App Update Mechanism

The app SHOULD use the Play Core In-App Updates API.

**Testable:** Verify In-App Updates API integration.
