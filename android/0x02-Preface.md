# Preface

The upstream OWASP MASVS is intentionally platform-agnostic — it defines *what* to verify, not *how* on a specific OS. This project takes each MASVS control and expands it into concrete, testable Android sub-requirements that reference specific APIs, manifest attributes, configuration files, and platform behaviors.

## Why Android-Specific Requirements?

Android's security model is unique. Hardware-backed keystores, the Intent IPC system, Network Security Configuration, scoped storage, BiometricPrompt, Play Integrity API, and dozens of other platform mechanisms require Android-specific guidance that a platform-agnostic standard cannot provide.

Developers, auditors, and pentesters need requirements they can directly act on — not abstract goals that require translation into platform-specific implementations.

## Scope

- Covers **Android 10 (API 29) through Android 17 (API 37)** with notes on version-specific behaviors
- References the **Android SDK, AndroidX/Jetpack, Google Play Services, and Google Tink** libraries
- Maps to **OWASP Mobile Top 10 2024**, **NIST SP 800-163**, and **CIS Android Benchmarks**
- Covers **native (Kotlin/Java)** apps; Flutter/React Native/Xamarin apps should also apply their framework-specific guidance

## Upstream Mapping

Every control in this document maps 1:1 to the upstream OWASP MASVS v2.x control of the same ID. Sub-requirements (e.g., `MASVS-STORAGE-1.3`) are Android-specific expansions that do not exist in the upstream standard.
