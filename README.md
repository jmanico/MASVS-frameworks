# OWASP MASVS Platform-Specific Frameworks

[![OWASP Incubator](https://img.shields.io/badge/owasp-incubator-blue.svg)](https://owasp.org/www-project-mobile-app-security/)
[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-sa/4.0/)

Platform-specific security verification requirements extending the [OWASP Mobile Application Security Verification Standard (MASVS)](https://github.com/OWASP/owasp-masvs). The upstream MASVS is intentionally platform-agnostic. This project provides concrete, testable sub-requirements for each mobile platform.

**This is NOT an official OWASP project.** It is a community resource maintained by [Jim Manico](https://github.com/jmanico) for teams that need platform-specific security requirements they can hand directly to developers, auditors, and pentesters.

## Platforms

| Platform | Status | Controls | Sub-Requirements |
|----------|--------|----------|-----------------|
| [Android](android/) | Active | 24 controls, 144 sub-requirements + 93 training-aligned requirements | Android 10 (API 29) through Android 17 (API 37) |
| [iOS](ios/) | Planned | Coming soon | iOS 16 through iOS 26 |

## How to Use

1. Start with the upstream [OWASP MASVS](https://mas.owasp.org/MASVS/) for platform-agnostic controls
2. Navigate to your platform directory for implementation-specific requirements
3. Use the sub-requirements as a testable checklist for development, audit, or pentest

## How to Contribute

1. Fork this repository
2. Create a feature branch
3. Submit a pull request with a clear description of the change
4. Reference the upstream MASVS control ID and any platform documentation links

## License

This work is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License](https://creativecommons.org/licenses/by-sa/4.0/).

Based on the [OWASP MASVS](https://github.com/OWASP/owasp-masvs) by the OWASP Foundation.
