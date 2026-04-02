# MASVS-STORAGE: Secure Data Storage at Rest

## Overview

iOS provides multiple storage mechanisms with different security properties. The Keychain is an encrypted database for passwords, keys, and certificates. The Data Protection API encrypts files with four protection classes based on device lock state. The Secure Enclave is a dedicated security coprocessor where keys never leave the hardware. This chapter covers secure use of these mechanisms.

## iOS Storage Landscape

| Storage Location | Security Level | Access | Notes |
|---|---|---|---|
| Keychain (ThisDeviceOnly) | Highest | App + shared access groups | Encrypted, hardware-protected, survives app reinstall |
| Keychain (default) | High | App + shared access groups | May sync via iCloud Keychain |
| Secure Enclave keys | Highest | App-only, hardware-bound | Keys never leave the coprocessor |
| Data Protection (Complete) | High | App-only when unlocked | Files inaccessible when device is locked |
| Data Protection (CompleteUnlessOpen) | High | App-only, open files persist | For background writes to already-open files |
| Data Protection (UntilFirstUnlock) | Medium | App-only after first unlock | Available in background after first unlock; default class |
| Core Data / SQLite | Low | App-only (plaintext .sqlite) | Encrypt with SQLCipher or Data Protection for sensitive data |
| UserDefaults | Low | App-only (plaintext plist) | Never for secrets |
| App container files | Low | App sandbox | Apply Data Protection classes for sensitive files |
| Shared container (App Groups) | Low | Apps in same group | Minimize sensitive data in shared containers |

## Key iOS APIs

- **Keychain Services** (Security framework) - Encrypted credential and key storage with accessibility classes
- **Data Protection API** - File-level encryption tied to device lock state (NSFileProtectionComplete, etc.)
- **Secure Enclave** (CryptoKit SecureEnclave.P256) - Hardware-bound key generation and operations
- **SQLCipher** - Transparent SQLite encryption for Core Data

## OWASP Mobile Top 10 2024 Mapping

- **M9 - Insecure Data Storage:** Directly addressed by MASVS-STORAGE-1
- **M1 - Improper Credential Usage:** Addressed by MASVS-STORAGE-1.1, 1.2, 1.5
- **M8 - Security Misconfiguration:** Addressed by MASVS-STORAGE-1.6, MASVS-STORAGE-2.1

---

## MASVS-STORAGE-1

### Control

The app securely stores sensitive data.

### Description

iOS apps handle sensitive data from many sources (user input, backend APIs, system services) and frequently need to persist it locally. iOS provides the Keychain for credentials, the Data Protection API for file encryption, and the Secure Enclave for hardware-bound keys. Make sure any sensitive data stored by the app uses the appropriate mechanism with correct accessibility classes and access controls.

### iOS Sub-Requirements

#### MASVS-STORAGE-1.1 - Use Keychain for Credentials and Tokens

Store all credentials, tokens, API keys, and certificates in the Keychain using `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`. Never store secrets in UserDefaults, plists, or the app bundle.

**Rationale:** The Keychain is an encrypted database protected by the device passcode and hardware keys. UserDefaults writes plaintext plist files that are trivially readable via backup extraction or on jailbroken devices.

#### MASVS-STORAGE-1.2 - Use ThisDeviceOnly Keychain Classes

Use `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` or `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` for sensitive items that should not sync to iCloud Keychain or transfer to new devices.

**Rationale:** Without the ThisDeviceOnly suffix, Keychain items may sync via iCloud Keychain to other devices on the same Apple ID, expanding the attack surface.

#### MASVS-STORAGE-1.3 - Use Data Protection Complete for Sensitive Files

Files containing sensitive data use `NSFileProtectionComplete` so they are inaccessible when the device is locked. Set this via file attributes or the entitlements default protection class.

**Rationale:** NSFileProtectionComplete encrypts files with a key derived from the device passcode. The key is evicted from memory when the device locks, making the files unreadable until the user unlocks.

#### MASVS-STORAGE-1.4 - Encrypt Databases Containing Sensitive Data

Core Data or SQLite databases containing sensitive data use SQLCipher or equivalent encryption. The encryption key is stored in the Keychain, not hardcoded.

**Rationale:** SQLite databases in the app container are plaintext files. On jailbroken devices or via backup extraction, their contents are readable with any SQLite client.

#### MASVS-STORAGE-1.5 - Never Store Secrets in UserDefaults or Plists

The app does not store sensitive values (tokens, passwords, API keys, PII) in UserDefaults, Info.plist, custom plist files, or bundled resources.

**Rationale:** UserDefaults writes to a plaintext plist in the app's Library/Preferences directory. These files are included in backups and trivially readable.

#### MASVS-STORAGE-1.6 - Exclude Sensitive Data from Backups

Files and directories containing sensitive data have the `isExcludedFromBackup` resource value set to true, preventing inclusion in iTunes/Finder and iCloud backups.

**Rationale:** iOS backups may be stored unencrypted on a computer. Sensitive files included in backups are accessible to anyone with access to the backup.

#### MASVS-STORAGE-1.7 - Protect Keychain Items with Biometric Access Control

Keychain items protecting high-value data use `SecAccessControl` with `.biometryCurrentSet` to require biometric authentication before access.

**Rationale:** Without biometric gating, any code running in the app's process can read the Keychain item. Biometric binding via .biometryCurrentSet also invalidates the item if biometric enrollment changes.

#### MASVS-STORAGE-1.8 - Use Secure Enclave Keys for High-Value Cryptographic Material

For signing keys, key agreement keys, or other high-value cryptographic material, the app generates keys inside the Secure Enclave using `SecureEnclave.P256`. These keys never leave the hardware.

**Rationale:** Secure Enclave keys cannot be extracted even on jailbroken devices. The private key material exists only inside the dedicated security coprocessor.

#### MASVS-STORAGE-1.9 - Clear Sensitive Data from Memory When No Longer Needed

The app zeros out sensitive data (passwords, keys, tokens) from memory after use. Use `Data` with `resetBytes(in:)` or `memset_s` for C buffers. Avoid storing secrets in Swift `String` (which is value-type but still managed by ARC).

**Rationale:** On jailbroken devices, memory dumps can extract sensitive values from the app's heap.

---

## MASVS-STORAGE-2

### Control

The app prevents leakage of sensitive data.

### Description

Sensitive data can be unintentionally exposed through iOS platform mechanisms such as system logs, screenshots, pasteboard, keyboard caches, notifications, Spotlight indexing, and backup files. These leaks happen as side effects of using standard APIs. Make sure developers actively prevent these exposures.

### iOS Sub-Requirements

#### MASVS-STORAGE-2.1 - No Sensitive Data in System Logs

The app does not write sensitive data to system logs via os_log, NSLog, or print. Release builds disable verbose logging.

**Rationale:** iOS logs persist in sysdiagnose bundles that users share in bug reports. Logs are also accessible on jailbroken devices and via Xcode console.

#### MASVS-STORAGE-2.2 - Prevent Screenshot Capture of Sensitive Screens

The app obscures sensitive content when entering the background by adding an overlay in `applicationWillResignActive(_:)` or `sceneWillResignActive(_:)` and removing it in `applicationDidBecomeActive(_:)`.

**Rationale:** iOS captures a screenshot of the app for the app switcher when backgrounding. Without protection, sensitive data appears in the app switcher thumbnail.

#### MASVS-STORAGE-2.3 - Disable Keyboard Cache for Sensitive Input Fields

Text fields accepting sensitive data use `secureTextEntry = true` for passwords and `autocorrectionType = .no` with `spellCheckingType = .no` for other sensitive fields to prevent keyboard caching.

**Rationale:** The iOS keyboard caches typed text for autocorrect and predictions. This cache persists on disk.

#### MASVS-STORAGE-2.4 - Prevent Pasteboard Leakage

The app avoids copying sensitive data to the system pasteboard. If copy is necessary, use `UIPasteboard.general.setItems(_:options:)` with `localOnly: true` and `expirationDate` to limit exposure. On iOS 16+, the system prompts the user before another app reads the pasteboard.

**Rationale:** The pasteboard is shared across apps. Before iOS 16, any foreground app could read it silently.

#### MASVS-STORAGE-2.5 - Mask Sensitive Data in Notifications

Notifications do not contain sensitive data in their visible content. Use `hiddenPreviewsBodyPlaceholder` for notification content that should be hidden on the lock screen.

**Rationale:** Notifications are visible on the lock screen, notification center, and connected Apple Watch.

#### MASVS-STORAGE-2.6 - Clear WebView Data After Sensitive Sessions

After loading sensitive content in WKWebView, the app clears cached data using `WKWebsiteDataStore.default().removeData(ofTypes:modifiedSince:completionHandler:)`.

**Rationale:** WKWebView caches, cookies, and local storage persist on the filesystem.

#### MASVS-STORAGE-2.7 - Exclude Sensitive Data from Spotlight Search Index

The app does not index sensitive data (credentials, financial information, health data) in Core Spotlight or NSUserActivity for search.

**Rationale:** Spotlight indexed data is accessible from the lock screen search and may appear in Siri suggestions.

#### MASVS-STORAGE-2.8 - Prevent Data Leakage via App Extensions

App extensions that handle sensitive data minimize the data shared via the shared container and clean up temporary data after use.

**Rationale:** Shared containers are accessible to all apps and extensions in the same App Group.

---

## Training-Aligned Requirements

iOS-specific requirements for MASVS-STORAGE-1 and MASVS-STORAGE-2.

### MASVS-STORAGE-1: Secure Storage of Sensitive Data

#### STORAGE-IOS-1.1: Keychain Accessibility Class Validation

The app MUST store all credentials and tokens in the Keychain with an appropriate accessibility class. `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` is required for high-sensitivity items.

**Testable:** Dump Keychain items on a jailbroken device. Verify accessibility class attributes.

#### STORAGE-IOS-1.2: Data Protection Class Verification

Files containing sensitive data MUST use NSFileProtectionComplete. The default data protection class for the app should be set to Complete in the entitlements.

**Testable:** Inspect file attributes on a jailbroken device while locked. Verify files are inaccessible.

#### STORAGE-IOS-1.3: Secure Enclave Key Usage

For apps processing financial data or sensitive credentials, signing keys SHOULD be generated in the Secure Enclave using SecureEnclave.P256.

**Testable:** Verify key generation uses CryptoKit Secure Enclave APIs. Verify keys cannot be exported.

#### STORAGE-IOS-1.4: Backup Exclusion

The app MUST set isExcludedFromBackup on directories containing sensitive data.

**Testable:** Perform an iTunes/Finder backup. Verify no sensitive data in the backup contents.

#### STORAGE-IOS-1.5: No Secrets in UserDefaults

The app MUST NOT store tokens, credentials, PII, or session identifiers in UserDefaults.

**Testable:** Inspect the app's plist files in Library/Preferences. Verify no sensitive data present.

### MASVS-STORAGE-2: Prevention of Sensitive Data Leakage

#### STORAGE-IOS-2.1: Log Sanitization

The app MUST NOT log sensitive data (tokens, credentials, PII) in release builds.

**Testable:** Run app with Console.app attached and exercise all features. No sensitive data in log output.

#### STORAGE-IOS-2.2: Screenshot Protection

Screens displaying sensitive data MUST obscure content when the app enters background.

**Testable:** Background the app on a sensitive screen. Check the app switcher thumbnail.

#### STORAGE-IOS-2.3: Pasteboard Protection

The app SHOULD NOT copy sensitive data to the system pasteboard. If necessary, use localOnly and expiration.

**Testable:** Monitor pasteboard contents during app use.
