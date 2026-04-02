# Contributing to MASVS-Android

Thank you for your interest in contributing to the OWASP MASVS for Android project.

## How to Contribute

### Reporting Issues

- Use GitHub Issues to report errors, suggest new sub-requirements, or request clarification
- Include the control ID (e.g., MASVS-STORAGE-1.3) in your issue title
- Reference specific Android API versions, classes, or documentation links

### Pull Requests

1. Fork the repository and create a feature branch
2. Make your changes following the existing format:
   - Chapter files (`0x1X-MASVS-*.md`) contain overview, controls, and training requirements
   - Each sub-requirement has an ID, title, rationale, and Android references
3. Update `0x00-Header.yaml` if adding/modifying control groups
4. Submit a PR with a clear description referencing the upstream MASVS control ID

### Style Guidelines

- Be specific: reference Android API names, class names, manifest attributes, and API levels
- Be testable: each sub-requirement should be verifiable by a developer or auditor
- Include rationale: explain *why* the requirement matters, not just *what* to do
- Reference Android documentation: link to developer.android.com where applicable
- Stay current: note API level requirements and deprecation status of APIs
- Follow the repo-wide [Requirements Style Guide](../REQUIREMENTS-STYLE.md) for normative language, version scoping, and training-aligned formatting expectations

### What We Need

- Reviews of existing sub-requirements for accuracy and completeness
- New sub-requirements for emerging Android security features
- Code examples (Kotlin/Java) for complex requirements
- Testing guidance for each sub-requirement
- Mappings to additional standards (NIST, CIS, PCI DSS)

## Code of Conduct

This project follows the [OWASP Code of Conduct](https://owasp.org/www-policy/operational/code-of-conduct).
