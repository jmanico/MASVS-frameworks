# MASVS-PRIVACY: Privacy

## Overview

iOS has led the industry in privacy enforcement. App Tracking Transparency (ATT, iOS 14.5+) requires user permission before cross-app tracking, with 75%+ of users opting out. Privacy Nutrition Labels on the App Store declare data practices before download. Link Tracking Protection (iOS 17+) strips tracking parameters from URLs. Locked and Hidden Apps (iOS 18) let users lock apps behind biometric authentication. Make sure apps practice data minimization, obtain proper consent, prevent tracking, and give users control over their data.

## iOS Privacy Evolution

| iOS Version | Key Privacy Feature |
|---|---|
| 14 | App Attest, local network permission prompt |
| 14.5 | App Tracking Transparency (ATT) |
| 15 | App Privacy Report, Mail Privacy Protection |
| 16 | Paste permission prompt, Safety Check, Lockdown Mode |
| 17 | Link Tracking Protection, Communication Safety expansion |
| 18 | Locked/Hidden Apps, reduced Contact Access, private Wi-Fi rotation, automatic passkey upgrades |
| 26 | Stolen Device Protection (default), Private Cloud Compute, accessory restriction, call screening |

## App Store Privacy Requirements

| Requirement | Description |
|---|---|
| Privacy Nutrition Labels | Declare data collected, linked to identity, used for tracking |
| ATT consent | Must prompt before cross-app tracking |
| Account deletion | Apps with account creation must support account deletion |
| Privacy policy | Required for all apps |

## OWASP Mobile Top 10 2024 Mapping

- **M6 - Inadequate Privacy Controls:** Directly addressed by MASVS-PRIVACY-1, MASVS-PRIVACY-2, MASVS-PRIVACY-3, MASVS-PRIVACY-4

---

## MASVS-PRIVACY-1

### Control

The app minimizes access to sensitive data and resources.

### Description

iOS apps should request only the data and permissions they need. The permission model has evolved from broad grants to granular, contextual requests with selected access. Make sure apps minimize data collection and that third-party SDKs respect user consent.

### iOS Sub-Requirements

#### MASVS-PRIVACY-1.1 - Request Only Necessary Permissions

Each permission has a meaningful `NSUsageDescription` in Info.plist. Permissions for optional features are requested when the feature is first used, not on launch.

**Rationale:** Excessive permissions erode trust and may trigger App Store rejection.

#### MASVS-PRIVACY-1.2 - Use the Most Restrictive Permission Available

Use approximate location instead of precise when sufficient. Use selected photos access (iOS 14+) instead of full library. Use Reduced Contact Access (iOS 18) when only specific contacts are needed.

**Rationale:** Least privilege limits data exposure.

#### MASVS-PRIVACY-1.3 - Handle Permission Denials Gracefully

The app continues functioning with degraded features when non-essential permissions are denied. The app provides context before requesting. The app respects denial without nagging. The app never blocks launch for non-critical permissions.

**Rationale:** Aggressive permission requests drive users to uninstall or distrust the app.

#### MASVS-PRIVACY-1.4 - Audit Third-Party SDK Data Collection

Every SDK is audited for what data it collects, whether it respects consent, whether it transmits to third parties, and whether it can be configured to minimize collection. All SDK data collection must be declared in Privacy Nutrition Labels.

**Rationale:** SDKs are the primary source of undisclosed data collection.

#### MASVS-PRIVACY-1.5 - Initialize SDKs Only After Consent

Analytics and advertising SDKs are not initialized until the user grants consent. SDKs that collect data before consent are removed or replaced.

**Rationale:** Pre-consent data collection violates ATT requirements and user trust.

#### MASVS-PRIVACY-1.6 - Minimize Background Data Access

The app does not access location, microphone, or camera in the background unless essential for core functionality. Background access is clearly disclosed.

**Rationale:** iOS shows indicators for background sensor access. Unexpected background access alarms users.

#### MASVS-PRIVACY-1.7 - Respect iOS 18 Locked/Hidden Apps

If the app handles sensitive content, it supports the Locked Apps feature and does not surface sensitive data via notifications or Siri when the app is locked.

**Rationale:** Users who lock apps behind biometric expect their content to be fully hidden.

---

## MASVS-PRIVACY-2

### Control

The app prevents identification of the user.

### Description

Protecting user identity and preventing cross-app tracking is critical on iOS, where Apple actively blocks tracking. Make sure the app uses privacy-preserving identifiers and respects ATT.

### iOS Sub-Requirements

#### MASVS-PRIVACY-2.1 - Respect App Tracking Transparency

The app requests ATT permission via `ATTrackingManager.requestTrackingAuthorization` before any cross-app tracking. If denied, the app does not use IDFA or alternative tracking identifiers.

**Rationale:** ATT is a legal and platform requirement. Circumventing it risks App Store rejection.

#### MASVS-PRIVACY-2.2 - Use Privacy-Preserving Identifiers

Use `identifierForVendor` for analytics (resets on uninstall of all apps from the same vendor). Use IDFA only with ATT consent for advertising. Do not use hardware identifiers. Use server-generated tokens for fraud detection.

**Rationale:** Each identifier type has a specific permitted use.

#### MASVS-PRIVACY-2.3 - Prevent Cross-Purpose Identifier Linking

Identifiers for one purpose (fraud detection) are not reused for another (advertising).

**Rationale:** Cross-purpose linking builds comprehensive user profiles.

#### MASVS-PRIVACY-2.4 - Anonymize Analytics Data

IP addresses are truncated, user IDs pseudonymized, location generalized, and event data aggregated.

**Rationale:** Minimizing PII in analytics reduces exposure from analytics provider breaches.

#### MASVS-PRIVACY-2.5 - Do Not Collect Device Fingerprinting Signals

The app does not collect combinations of device attributes (screen resolution, fonts, timezone, battery, sensors) for fingerprinting.

**Rationale:** Apple prohibits device fingerprinting and will reject apps that do it.

---

## MASVS-PRIVACY-3

### Control

The app is transparent about data collection and usage.

### Description

Apple requires Privacy Nutrition Labels declaring all data practices. Make sure disclosures are accurate and current.

### iOS Sub-Requirements

#### MASVS-PRIVACY-3.1 - Accurate Privacy Nutrition Labels

The app's App Store privacy labels accurately declare all data types collected, whether linked to identity, used for tracking, or collected by SDKs.

**Rationale:** Inaccurate labels violate App Store policy and user trust.

#### MASVS-PRIVACY-3.2 - In-App Privacy Disclosure

The app shows a privacy notice or consent dialog before first data collection explaining what data is collected, why it is collected, who receives it, how long it is retained, and how to exercise data rights.

**Rationale:** Users must understand data practices before collection begins.

#### MASVS-PRIVACY-3.3 - Disclose Non-Obvious Data Collection

Background data collection, SDK data collection, pre-engagement collection, and collection from non-primary sources (clipboard, contacts, calendar) are explicitly disclosed.

**Rationale:** Non-obvious collection is the primary source of privacy complaints and regulatory action.

#### MASVS-PRIVACY-3.4 - iOS 17+: Link Tracking Protection Awareness

The app accounts for Link Tracking Protection (Mail, Messages, and Safari strip tracking parameters like `fbclid`, `gclid`). Attribution should not depend solely on URL tracking parameters.

**Rationale:** iOS 17 actively strips tracking parameters, breaking attribution that relies on them.

#### MASVS-PRIVACY-3.5 - Keep Disclosures Current

Privacy labels and in-app disclosures are updated whenever data practices change (new SDKs, new data types, new features).

**Rationale:** Stale disclosures create compliance gaps and mislead users.

---

## MASVS-PRIVACY-4

### Control

The app offers user control over their data.

### Description

Users must be able to delete their data, manage consent, and export their information. Apple requires account deletion for apps with account creation.

### iOS Sub-Requirements

#### MASVS-PRIVACY-4.1 - Provide Account and Data Deletion

The app provides account deletion accessible from within the app and from a web URL. Deletion removes all server-side personal data. Deletion is processed within a documented timeframe. Apple requires this for all apps with account creation.

**Rationale:** App Store requirement and privacy regulation compliance.

#### MASVS-PRIVACY-4.2 - Allow Granular Consent Management

The user can grant or deny consent per purpose (analytics, advertising, personalization). Consent choices can be modified at any time. Withdrawal is as easy as granting.

**Rationale:** Bundled consent does not meet GDPR or best practice standards.

#### MASVS-PRIVACY-4.3 - Respect Consent Withdrawal

When consent is revoked: data collection stops immediately, SDKs are reconfigured, previously collected data is deleted or anonymized, and changes take effect without app restart.

**Rationale:** Delayed or incomplete consent withdrawal undermines the consent model.

#### MASVS-PRIVACY-4.4 - Support Data Export

For apps subject to GDPR or similar regulations, the app provides data export in a machine-readable format (JSON, CSV).

**Rationale:** Data portability is a regulatory requirement under GDPR Article 20 and similar laws.

#### MASVS-PRIVACY-4.5 - Re-Prompt on Expanded Data Collection

If the app begins collecting new data types after an update, it re-prompts for consent before the new collection begins.

**Rationale:** Initial consent does not cover future unrelated uses.

#### MASVS-PRIVACY-4.6 - Respect System Privacy Controls

The app respects permission revocation in Settings. The app does not circumvent permission denial (e.g., using WiFi info when location is denied). The app handles authorization status changes gracefully.

**Rationale:** System-level controls represent the user's explicit privacy intent.

---

## Training-Aligned Requirements

| ID | Requirement | Verification |
|---|---|---|
| PRIVACY-IOS-1.1 | **ATT Implementation** - The app MUST use `ATTrackingManager` before any cross-app tracking. | Opt out of tracking. Verify no IDFA transmitted. |
| PRIVACY-IOS-1.2 | **Minimal Permissions** - The app MUST request only necessary permissions with `NSUsageDescription`. | List all declared permissions in Info.plist and verify each is used. |
| PRIVACY-IOS-1.3 | **SDK Consent Gating** - SDKs MUST NOT collect data before consent. | Monitor network traffic before consent is granted. |
| PRIVACY-IOS-2.1 | **Privacy Nutrition Labels** - Labels MUST accurately reflect actual data practices. | Compare declared labels vs actual data collection behavior. |
| PRIVACY-IOS-2.2 | **No Fingerprinting** - The app MUST NOT use device fingerprinting. | Monitor data collection for fingerprinting signals. |
| PRIVACY-IOS-3.1 | **Account Deletion** - The app MUST provide account and data deletion. | Delete account, verify server-side data removal. |
| PRIVACY-IOS-3.2 | **Local Data Cleanup** - On account deletion or logout, the app MUST remove Keychain entries, files, databases, caches, and cookies. | Inspect local storage after logout. |
| PRIVACY-IOS-3.3 | **Data Export** - The app SHOULD provide data export in a portable format. | Request export, verify format (JSON or CSV). |
