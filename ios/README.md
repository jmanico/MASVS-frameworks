# OWASP MASVS for iOS

iOS-specific security verification requirements extending [OWASP MASVS v2.1.0](https://mas.owasp.org/MASVS/).

## Status

**Planned.** iOS-specific requirements are under development. Target coverage: iOS 16 through iOS 26.

Expected control groups (mirroring MASVS):

| # | Group | Focus |
|---|---|---|
| 1 | MASVS-STORAGE | Keychain, Secure Enclave, Data Protection API |
| 2 | MASVS-CRYPTO | CryptoKit, Secure Enclave P256, quantum-secure TLS (iOS 26) |
| 3 | MASVS-AUTH | LAContext, biometrics (.biometryCurrentSet), passkeys (iOS 16+) |
| 4 | MASVS-NETWORK | App Transport Security, TLS enforcement |
| 5 | MASVS-PLATFORM | Universal Links, URL schemes, App Groups, WKWebView |
| 6 | MASVS-CODE | Code signing, provisioning profiles, Xcode Managed Signing |
| 7 | MASVS-RESILIENCE | App Attest, DeviceCheck, jailbreak detection |
| 8 | MASVS-PRIVACY | ATT, Privacy Nutrition Labels, Locked/Hidden Apps (iOS 18) |

## How to Contribute

Contributions welcome. See the [main README](../README.md) for contribution guidelines.
