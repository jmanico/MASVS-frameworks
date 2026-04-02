# Appendix A: Glossary

| Term | Definition |
|---|---|
| **AASA** | Apple App Site Association. A JSON file hosted at `/.well-known/apple-app-site-association` that proves domain ownership for Universal Links |
| **App Attest** | DCAppAttestService. Hardware-backed attestation service (iOS 14+) that proves app integrity to a server using a key generated in the Secure Enclave |
| **ATS** | App Transport Security. iOS enforcement mechanism requiring TLS 1.2+ with forward secrecy for all network connections |
| **ATT** | App Tracking Transparency. iOS 14.5+ framework requiring user permission before cross-app tracking |
| **CryptoKit** | Apple's Swift-native cryptographic framework providing AES-GCM, ChaChaPoly, P256 ECDSA, HKDF, and Secure Enclave key operations |
| **CXP** | FIDO Credential Exchange Protocol. Used for passkey import/export across platforms |
| **Data Protection API** | iOS file encryption system with four protection classes based on device lock state |
| **DeviceCheck** | Apple service providing 2 bits of server-side per-device state for fraud detection and trial tracking |
| **FIDO2** | Fast Identity Online 2. The standard behind passkeys/WebAuthn for phishing-resistant authentication |
| **Keychain** | iOS encrypted database for storing passwords, keys, certificates, and other small secrets. Protected by Data Protection and access control policies |
| **LAContext** | LocalAuthentication framework context object used for biometric (Face ID/Touch ID) and device credential authentication |
| **Lockdown Mode** | iOS 16+ extreme hardening mode for at-risk users. Disables JIT, message attachments, wired connections, and configuration profiles |
| **MASVS** | Mobile Application Security Verification Standard. OWASP's standard for mobile app security requirements |
| **NSFileProtectionComplete** | Data Protection class where files are inaccessible when the device is locked. Strongest file protection available |
| **PAC** | Pointer Authentication Codes. Hardware-enforced control flow integrity on A12+ chips that prevents ROP/JOP exploit chains |
| **Passkey** | FIDO2 discoverable credential synced via iCloud Keychain. Phishing-resistant, passwordless authentication (iOS 16+) |
| **PKCE** | Proof Key for Code Exchange. Extension to OAuth 2.0 authorization code flow to prevent code interception attacks |
| **Private Cloud Compute** | Apple's attested server-side AI processing in Apple silicon. Data processed but never stored or accessible to Apple |
| **SBOM** | Software Bill of Materials. A list of all software components, versions, and dependencies in an application |
| **Secure Enclave** | Dedicated security coprocessor with its own boot ROM and AES engine. Handles biometric matching, key storage, and cryptographic operations. Keys never leave the hardware |
| **SecureEnclave.P256** | CryptoKit API for generating and using ECDSA P-256 keys inside the Secure Enclave |
| **Universal Links** | iOS verified deep links using `https://` scheme with AASA file verification. Only the verified app can handle the link |
| **WKWebView** | Modern WebView with process isolation (runs in a separate process from the app). Replaced UIWebView (deprecated iOS 12) |
