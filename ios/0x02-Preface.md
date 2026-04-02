# Preface

The upstream OWASP MASVS is intentionally platform-agnostic. It defines *what* to verify, not *how* on a specific OS. This project takes each MASVS control and expands it into concrete, testable iOS sub-requirements that reference specific frameworks, APIs, entitlements, and platform behaviors.

## Why iOS-Specific Requirements?

iOS has a unique security architecture built on hardware roots of trust. The Secure Enclave, Data Protection API, Keychain Services, App Transport Security, CryptoKit, App Attest, and dozens of other platform mechanisms require iOS-specific guidance that a platform-agnostic standard cannot provide.

Developers, auditors, and pentesters need requirements they can directly act on, not abstract goals that require translation into platform-specific implementations.

## Scope

- Covers **iOS 16 through iOS 26** with notes on version-specific behaviors
- References **UIKit, SwiftUI, CryptoKit, AuthenticationServices, LocalAuthentication, and Apple platform frameworks**
- Maps to **OWASP Mobile Top 10 2024**, **NIST SP 800-163**, and **Apple Platform Security Guide**
- Covers **native (Swift/Objective-C)** apps; Flutter/React Native/Xamarin apps should also apply their framework-specific guidance

## Upstream Mapping

Every control in this document maps 1:1 to the upstream OWASP MASVS v2.x control of the same ID. Sub-requirements (e.g., `MASVS-STORAGE-1.3`) are iOS-specific expansions that do not exist in the upstream standard.
