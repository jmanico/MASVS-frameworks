# Appendix A: Glossary

| Term | Definition |
|---|---|
| **AEAD** | Authenticated Encryption with Associated Data — encryption that provides both confidentiality and integrity (e.g., AES-GCM) |
| **AES-GCM** | Advanced Encryption Standard in Galois/Counter Mode — the recommended symmetric encryption mode for Android |
| **AndroidKeyStore** | Android's system-level keystore backed by TEE or StrongBox hardware; keys never enter app process memory |
| **App Links** | Android's verified deep links using `https://` scheme with Digital Asset Links (`.well-known/assetlinks.json`) verification |
| **BiometricPrompt** | AndroidX API for biometric authentication supporting Class 3 (strong) biometrics with CryptoObject binding |
| **CryptoObject** | A wrapper for `Cipher`, `Signature`, or `Mac` objects used with BiometricPrompt to bind biometric authentication to a cryptographic operation |
| **CT** | Certificate Transparency — a system for publicly logging TLS certificates to detect misissued or rogue certificates |
| **Custom Tab** | Chrome Custom Tab (`CustomTabsIntent`) — system browser integration for OAuth flows per RFC 8252 |
| **DataStore** | Jetpack library for key-value and proto-based data storage, replacing SharedPreferences; supports Tink encryption |
| **DEX** | Dalvik Executable — the bytecode format used by Android Runtime (ART); contained in APK/AAB bundles |
| **DPoP** | Demonstration of Proof-of-Possession (RFC 9449) — sender-constrained access tokens bound to a cryptographic key |
| **FBE** | File-Based Encryption — Android's filesystem encryption (default since Android 10) |
| **FIDO2** | Fast Identity Online 2 — the standard behind passkeys/WebAuthn for phishing-resistant authentication |
| **FLAG_SECURE** | `WindowManager.LayoutParams.FLAG_SECURE` — prevents screenshots, screen recording, and non-secure display output |
| **Frida** | A dynamic instrumentation toolkit used for reverse engineering and security testing of mobile apps |
| **HPKE** | Hybrid Public Key Encryption (RFC 9180) — post-quantum-ready key encapsulation mechanism |
| **IPC** | Inter-Process Communication — Android mechanisms for communication between apps (Intents, Content Providers, Broadcast Receivers, Services) |
| **Keystore** | See AndroidKeyStore |
| **MASVS** | Mobile Application Security Verification Standard — OWASP's standard for mobile app security requirements |
| **Network Security Configuration** | Android's declarative XML-based system (`network_security_config.xml`) for configuring TLS policy, certificate pinning, and trust anchors |
| **NSC** | Network Security Configuration |
| **Passkey** | FIDO2 discoverable credential synced via Google Password Manager; phishing-resistant, passwordless authentication |
| **PKCE** | Proof Key for Code Exchange — extension to OAuth 2.0 authorization code flow to prevent code interception attacks |
| **Play Integrity API** | Google Play API for device attestation, app integrity verification, and licensing checks; verdicts must be evaluated server-side |
| **R8** | Google's code shrinker and obfuscator for Android, replacing ProGuard; performs identifier renaming, dead code removal, and optimization |
| **RASP** | Runtime Application Self-Protection — SDKs that provide continuous runtime threat detection |
| **SBOM** | Software Bill of Materials — a list of all software components, versions, and dependencies in an application |
| **Scoped Storage** | Android's storage model (enforced since API 30) restricting app access to app-specific directories and user-selected files |
| **SharedPreferences** | Android's key-value storage mechanism; stores data as plaintext XML in internal storage |
| **SPKI** | SubjectPublicKeyInfo — the portion of a certificate containing the public key; used for certificate pinning |
| **SQLCipher** | Open-source transparent encryption for SQLite databases; compatible with Android Room |
| **StrongBox** | A dedicated secure processor (physically separate from the main SoC) for cryptographic key storage; FIPS 140-2 Level 3 |
| **TEE** | Trusted Execution Environment — a secure area of the main processor (e.g., ARM TrustZone) for cryptographic operations |
| **Tink** | Google's multi-language cryptographic library; recommended for Android AEAD, streaming encryption, and key management |
| **WebAuthn** | Web Authentication API — the W3C standard for passwordless authentication using public-key cryptography |
| **Xposed** | A framework for modifying Android app behavior at runtime by hooking Java methods; LSPosed is the modern successor |
| **ZipSlip** | A path traversal vulnerability in ZIP archive extraction where crafted entry names (containing `../`) overwrite files outside the target directory |
