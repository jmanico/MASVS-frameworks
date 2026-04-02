# MASVS-PRIVACY: Privacy

## Overview

Android's privacy landscape has evolved dramatically, from the early days of broad permissions and unlimited identifier access to modern granular runtime permissions, scoped storage, restricted identifiers, privacy indicators, and the Privacy Sandbox. Make sure Android apps practice data minimization, use privacy-preserving identifiers, provide transparent disclosures, and give users meaningful control over their data.

## Android Privacy Evolution

| Android Version | Key Privacy Feature |
|---|---|
| 6.0 (API 23) | Runtime permissions |
| 8.0 (API 26) | `ANDROID_ID` scoped per-app/user; background execution limits |
| 10 (API 29) | Background location separate; scoped storage; MAC randomization; no hardware ID access |
| 11 (API 30) | One-time permissions; auto-revoke unused permissions; package visibility restrictions |
| 12 (API 31) | Approximate location option; clipboard access notification; privacy indicators (mic/camera) |
| 13 (API 33) | Granular media permissions; per-app language; notification permission; clipboard sensitive flag |
| 14 (API 34) | Partial photo/video selection; selected photos access |
| 15 (API 35) | Screen recording detection; Private Space; partial screen sharing |
| 16 (API 36) | Local network permission; app-specific MediaStore versions |

## Google Play Data Safety Section

| Declaration | Description |
|---|---|
| Data types collected | Location, contacts, financial, health, messages, photos, identifiers, etc. |
| Data shared | Which data types are shared with third parties |
| Security practices | Encryption in transit, data deletion mechanism |
| Third-party SDK data | All SDK data collection must be declared |
| Data deletion URL | Web-accessible deletion mechanism (required since Dec 2023) |

## Privacy Sandbox on Android

| API | Purpose | Status |
|---|---|---|
| Topics API | Interest-based advertising without cross-app tracking | GA |
| Attribution Reporting | Conversion measurement without cross-app identity | GA |
| Protected Audiences | On-device ad auctions | GA |
| SDK Runtime | Isolated SDK execution environment | Beta |

## OWASP Mobile Top 10 2024 Mapping

- **M6 - Inadequate Privacy Controls:** Directly addressed by all four controls

---

## MASVS-PRIVACY-1

### Control

The app minimizes access to sensitive data and resources.

### Description

Android apps should only request access to the data and system resources they absolutely need. Android's permission model has evolved significantly, from install-time grants to runtime permissions, one-time permissions, partial grants, and auto-revocation. Make sure apps practice data minimization, request only necessary permissions, and enforce that third-party SDKs respect the same constraints.

### Android Sub-Requirements

#### MASVS-PRIVACY-1.1 - Request Only Necessary Permissions

The app requests only the permissions that are strictly necessary for its core functionality. Every permission declared in `AndroidManifest.xml` has documented justification. Permissions that are only needed for optional features are requested when the feature is first used, not on app launch.

**Rationale:** Excessive permissions increase the attack surface and erode user trust.

#### MASVS-PRIVACY-1.2 - Use the Most Restrictive Permission Available

When multiple permission levels can achieve the same goal, the app uses the least privileged option: `ACCESS_COARSE_LOCATION` instead of `ACCESS_FINE_LOCATION` when approximate location suffices; `READ_MEDIA_IMAGES` (API 33+) instead of `READ_EXTERNAL_STORAGE`; `QUERY_ALL_PACKAGES` is avoided.

**Android References:**
- Android 12+ approximate location
- Android 13+ granular media permissions
- Android 14+ partial photo/video access

#### MASVS-PRIVACY-1.3 - Handle Permission Denials Gracefully

The app handles permission denials gracefully: Continues functioning with degraded features, Clearly explains why a permission is needed, Respects "Don't ask again", Never blocks app launch solely due to a denied non-critical permission.

#### MASVS-PRIVACY-1.4 - Audit Third-Party SDK Data Collection

Every third-party SDK is audited for what data it collects, whether it respects consent signals, whether it transmits data to third-party servers, whether it can be configured to minimize collection.

**Rationale:** Third-party SDKs are the primary source of undisclosed data collection.

#### MASVS-PRIVACY-1.5 - Enforce SDK Consent Compliance

Third-party SDKs are initialized only after the user has granted consent.

**Android References:**
- Initialize analytics/advertising SDKs only after consent flow completes

#### MASVS-PRIVACY-1.6 - Implement Scoped Storage Correctly

The app uses scoped storage (enforced since API 30): App-specific files, Shared media via `MediaStore`, User-selected files via Storage Access Framework, No `MANAGE_EXTERNAL_STORAGE` unless file manager.

#### MASVS-PRIVACY-1.7 - Minimize Background Data Access

The app does not access sensitive data in the background unless essential.

**Android References:**
- `ACCESS_BACKGROUND_LOCATION`
- Privacy indicators
- Background process restrictions

---

## MASVS-PRIVACY-2

### Control

The app prevents identification of the user.

### Description

Protecting user identity and preventing cross-app or cross-session tracking is a core privacy requirement. Make sure the app uses privacy-preserving identifiers.

### Android Sub-Requirements

#### MASVS-PRIVACY-2.1 - Do Not Use Hardware Device Identifiers

The app does not access or transmit hardware device identifiers: IMEI (removed since API 29), MEID, Device serial number, MAC addresses (randomized since API 29), ANDROID_ID (scoped since API 26).

**Rationale:** Hardware identifiers are permanent and enable cross-app tracking.

#### MASVS-PRIVACY-2.2 - Use Privacy-Preserving Identifiers

The app uses the appropriate identifier for each use case: Advertising (Google Advertising ID or Topics API), Analytics (app-instance ID), Push notifications (FCM token), Fraud detection (server-generated tokens).

#### MASVS-PRIVACY-2.3 - Prevent Cross-Purpose Identifier Linking

Identifiers collected for one purpose are not reused for another purpose.

**Rationale:** Cross-purpose linking enables building comprehensive user profiles.

#### MASVS-PRIVACY-2.4 - Implement Data Anonymization for Analytics

Analytics data is anonymized: IP addresses truncated, User IDs pseudonymized, Location data generalized, Event data aggregated.

#### MASVS-PRIVACY-2.5 - Minimize Device Fingerprinting Signals

The app does not collect or transmit combinations of device attributes for fingerprinting.

**Android References:**
- Android 16 app-specific MediaStore versions
- Package visibility restrictions (API 30+)

---

## MASVS-PRIVACY-3

### Control

The app is transparent about data collection and usage.

### Description

Users need clear visibility into how their data is collected, stored, processed, and shared. This section combines Android-facing transparency controls with closely related Google Play disclosure obligations.

### Android Sub-Requirements

#### MASVS-PRIVACY-3.1 - Complete and Accurate Google Play Data Safety Section

The app's Google Play Data Safety Section accurately declares all data types, purposes, encryption practices, deletion capabilities, and all third-party SDK data collection.

**Rationale:** Google Play Data Safety is the primary transparency mechanism.

#### MASVS-PRIVACY-3.2 - Provide In-App Privacy Disclosure

The app displays a clear privacy notice on first launch explaining what data is collected, sharing, retention, and data rights.

#### MASVS-PRIVACY-3.3 - Disclose Non-Obvious Data Collection

Any data collection behavior that a reasonable user would not expect is explicitly disclosed: Background collection, non-primary input sources, pre-engagement collection, SDK data collection.

#### MASVS-PRIVACY-3.4 - Display Privacy Indicators and Notifications

The app respects Android's built-in privacy indicators and provides its own in-app indicators. Uses `ForegroundServiceType` declarations.

**Android References:**
- Android 12+ green status bar dots

#### MASVS-PRIVACY-3.5 - Keep Disclosures Current

Privacy disclosures are updated whenever new data types are collected, new SDKs are added, sharing practices change, or new features access sensitive data.

---

## MASVS-PRIVACY-4

### Control

The app offers user control over their data.

### Description

Users should have meaningful control over their personal data. Make sure the app provides data management capabilities in compliance with privacy regulations.

### Android Sub-Requirements

#### MASVS-PRIVACY-4.1 - Provide Data Deletion Capability

Where the product offers accounts or stores personal data beyond what is required for immediate app operation, the app or associated service provides a mechanism for users to request deletion of their account and associated data. When Google Play policy or applicable regulation requires it, this capability should be accessible from within the app and from a web URL.

**Android References:**
- Google Play policy requires a data deletion URL

#### MASVS-PRIVACY-4.2 - Allow Granular Consent Management

Where the product relies on consent-based collection or sharing, the app provides a consent management interface where the user can grant or deny consent per purpose, modify choices, and withdraw consent.

#### MASVS-PRIVACY-4.3 - Respect Consent Withdrawal

When a user revokes consent: The app immediately stops the associated data collection, Third-party SDKs are reconfigured, Previously collected data is deleted or anonymized, Changes take effect without requiring restart.

#### MASVS-PRIVACY-4.4 - Support Data Export / Portability

For apps subject to GDPR, contractual portability requirements, or similar obligations, the app or associated service provides data export in a machine-readable format such as JSON or CSV.

#### MASVS-PRIVACY-4.5 - Re-Prompt on Expanded Data Collection

If the app begins collecting new data types, it re-prompts for consent.

**Rationale:** Initial consent does not cover future unrelated uses.

#### MASVS-PRIVACY-4.6 - Integrate with Android Permission Settings

The app respects system-level privacy controls: Honors permission revocation, Handles auto-revoked permissions gracefully, Does not use workarounds to circumvent denial.

**Android References:**
- Auto-revoke (API 30+)
- `ActivityCompat.shouldShowRequestPermissionRationale()`

---

## Training-Aligned Requirements

Android-specific requirements for MASVS-PRIVACY-1 through MASVS-PRIVACY-4.

### MASVS-PRIVACY-1: Data Minimization

#### PRIVACY-ANDROID-1.1: Scoped Storage Compliance

The app MUST use Scoped Storage and MUST NOT request `MANAGE_EXTERNAL_STORAGE` unless the app is a file manager.

**Testable:** Verify no `MANAGE_EXTERNAL_STORAGE` permission.

#### PRIVACY-ANDROID-1.2: Approximate Location

The app SHOULD use `ACCESS_COARSE_LOCATION` instead of `ACCESS_FINE_LOCATION` when exact coordinates are not required.

**Testable:** Review location permission requests.

#### PRIVACY-ANDROID-1.3: Selected Photos Access

On Android 14+, the app SHOULD use selected photos access.

**Testable:** Verify photo picker or selected access API usage.

#### PRIVACY-ANDROID-1.4: Minimal Permission Requests

The app MUST request only permissions necessary for core functionality. Each permission MUST have documented justification.

**Testable:** List all declared permissions.

### MASVS-PRIVACY-2: Granular User Consent

#### PRIVACY-ANDROID-2.1: Consent Before Collection

The app MUST obtain explicit user consent before collecting personal data. Consent MUST be granular.

**Testable:** Install app fresh. Verify consent UI.

#### PRIVACY-ANDROID-2.2: Play Data Safety Section

The app MUST maintain an accurate Google Play Data Safety section.

**Testable:** Compare declared practices with actual behavior.

#### PRIVACY-ANDROID-2.3: SDK Consent Gating

Third-party SDKs MUST NOT collect data before user consent is confirmed.

**Testable:** Monitor network traffic before consent.

### MASVS-PRIVACY-3: Anti-Tracking

#### PRIVACY-ANDROID-3.1: Advertising ID Restrictions

The app MUST respect the user's choice to opt out.

**Testable:** Disable personalized ads. Verify no alternative tracking.

#### PRIVACY-ANDROID-3.2: No Fingerprinting

The app MUST NOT use device fingerprinting techniques.

**Testable:** Monitor data collection.

#### PRIVACY-ANDROID-3.3: Privacy Dashboard Compliance

On Android 12+, the app's permission usage MUST be consistent with declared purposes.

**Testable:** Review Privacy Dashboard entries.

### MASVS-PRIVACY-4: Data Deletion

#### PRIVACY-ANDROID-4.1: Account and Data Deletion

The app MUST provide account deletion capability: Accessible within the app, Complete, Timely.

**Testable:** Request account deletion.

#### PRIVACY-ANDROID-4.2: Local Data Cleanup

On account deletion or logout, the app MUST remove all locally stored personal data: Keystore entries, Encrypted storage files, Database records, Cached data, Cookies.

**Testable:** Delete account. Inspect local storage.

#### PRIVACY-ANDROID-4.3: Data Export

The app SHOULD provide data export in a portable format.

**Testable:** Request data export.
