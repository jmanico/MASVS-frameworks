# Using MASVS for Android

## Target Audience

| Role | How to Use This |
|---|---|
| **Android Developers** | Implement each sub-requirement using the referenced APIs and configurations |
| **Security Auditors** | Use sub-requirements as a checklist; each is testable and maps back to upstream MASVS |
| **Pentesters** | Focus testing on the specific Android attack surfaces called out in each control |
| **Engineering Managers** | Use as acceptance criteria for security stories and compliance evidence |

## Document Structure

This standard is organized into eight control groups, each in its own chapter file:

| Chapter | Group | Focus |
|---|---|---|
| 0x10 | MASVS-STORAGE | Secure data storage at rest |
| 0x11 | MASVS-CRYPTO | Cryptography and key management |
| 0x12 | MASVS-AUTH | Authentication and authorization |
| 0x13 | MASVS-NETWORK | Network communication security |
| 0x14 | MASVS-PLATFORM | Platform interaction and IPC |
| 0x15 | MASVS-CODE | Code quality and supply chain |
| 0x16 | MASVS-RESILIENCE | Anti-tampering and anti-reverse engineering |
| 0x17 | MASVS-PRIVACY | Privacy and data minimization |

Each chapter contains three sections:

1. **Overview** - landscape, architecture diagrams, and API reference tables
2. **Controls** - detailed sub-requirements with rationale and Android references
3. **Training-Aligned Requirements** - additional testable requirements derived from OWASP/Manicode mobile security training

## How to Contribute

1. Fork this repository
2. Create a feature branch
3. Submit a pull request with a clear description of the change
4. Reference the upstream MASVS control ID and any Android documentation links

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed guidelines.
