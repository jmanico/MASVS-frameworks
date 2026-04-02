# OWASP MASVS for Android

[![OWASP Incubator](https://img.shields.io/badge/owasp-incubator-blue.svg)](https://owasp.org/www-project-mobile-app-security/)

**An Android-specific adaptation of the [OWASP Mobile Application Security Verification Standard (MASVS)](https://github.com/OWASP/owasp-masvs)** with detailed, platform-native requirements, API-level guidance, and implementation references.

**This is NOT an official OWASP project.** It is a community fork maintained by [Jim Manico](https://github.com/jmanico) for teams that need Android-specific security requirements they can hand directly to developers, auditors, and pentesters.

## Document Structure

| File | Content |
|---|---|
| [0x00-Header.yaml](0x00-Header.yaml) | Metadata and chapter index |
| [0x01-Frontispiece.md](0x01-Frontispiece.md) | About, copyright, acknowledgments |
| [0x02-Preface.md](0x02-Preface.md) | Why Android-specific requirements, scope, upstream mapping |
| [0x03-Using-MASVS-Android.md](0x03-Using-MASVS-Android.md) | Target audience, how to use, how to contribute |
| [0x10-MASVS-STORAGE.md](0x10-MASVS-STORAGE.md) | Secure data storage at rest |
| [0x11-MASVS-CRYPTO.md](0x11-MASVS-CRYPTO.md) | Cryptography and key management |
| [0x12-MASVS-AUTH.md](0x12-MASVS-AUTH.md) | Authentication and authorization |
| [0x13-MASVS-NETWORK.md](0x13-MASVS-NETWORK.md) | Network communication security |
| [0x14-MASVS-PLATFORM.md](0x14-MASVS-PLATFORM.md) | Platform interaction and IPC |
| [0x15-MASVS-CODE.md](0x15-MASVS-CODE.md) | Code quality and supply chain |
| [0x16-MASVS-RESILIENCE.md](0x16-MASVS-RESILIENCE.md) | Anti-tampering and anti-reverse engineering |
| [0x17-MASVS-PRIVACY.md](0x17-MASVS-PRIVACY.md) | Privacy and data minimization |
| [0x90-Appendix-A_Glossary.md](0x90-Appendix-A_Glossary.md) | Glossary of terms |
| [0x91-Appendix-B_References.md](0x91-Appendix-B_References.md) | References and external resources |

## Scope

- Covers **Android 10 (API 29) through Android 17 (API 37)** with notes on version-specific behaviors
- References the **Android SDK, AndroidX/Jetpack, Google Play Services, and Google Tink** libraries
- Maps to **OWASP Mobile Top 10 2024**, **NIST SP 800-163**, and **CIS Android Benchmarks**
- Covers **native (Kotlin/Java)** apps; Flutter/React Native/Xamarin apps should also apply their framework-specific guidance

## Control Groups

| # | Group | Focus |
|---|---|---|
| 0x10 | MASVS-STORAGE | Secure data storage at rest |
| 0x11 | MASVS-CRYPTO | Cryptography and key management |
| 0x12 | MASVS-AUTH | Authentication and authorization |
| 0x13 | MASVS-NETWORK | Network communication security |
| 0x14 | MASVS-PLATFORM | Platform interaction and IPC |
| 0x15 | MASVS-CODE | Code quality and supply chain |
| 0x16 | MASVS-RESILIENCE | Anti-tampering and anti-reverse engineering |
| 0x17 | MASVS-PRIVACY | Privacy and data minimization |

Each chapter contains:
1. **Overview** - landscape, architecture diagrams, and API reference tables
2. **Controls** - detailed sub-requirements with rationale and Android references
3. **Training-Aligned Requirements** - additional testable requirements from OWASP/Manicode training, currently presented as subsections, tables, or structured lists depending on the chapter

## Upstream Mapping

Every control maps 1:1 to the upstream OWASP MASVS v2.x control of the same ID. Sub-requirements (e.g., `MASVS-STORAGE-1.3`) are Android-specific expansions that do not exist in the upstream standard.

## How to Contribute

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Acknowledgments

- [OWASP Mobile Application Security](https://mas.owasp.org/) project team
- [Android Security documentation](https://developer.android.com/privacy-and-security)
- [OWASP Mobile Top 10 2024](https://owasp.org/www-project-mobile-top-10/)
