# MASVS-STORAGE: Secure Data Storage at Rest

## Overview

Mobile apps frequently store data locally on the device. Android provides multiple storage mechanisms - internal storage (app-sandboxed), external/shared storage, SharedPreferences, SQLite databases, the AndroidKeyStore, and file-based encryption. Each has different security properties. Make sure sensitive data is stored using the most appropriate mechanism with proper encryption and access controls.

## Android Storage Landscape

| Storage Location | Security Level | Access | Notes |
|---|---|---|---|
| Internal storage (`getFilesDir()`) | High | App-only (UID sandbox) | Best for sensitive data; encrypted at rest on FBE devices |
| No-backup files (`getNoBackupFilesDir()`) | High | App-only, excluded from backup | For sensitive data that should not be backed up |
| AndroidKeyStore | Highest | App-only, hardware-backed | For cryptographic keys; non-extractable |
| SharedPreferences | Medium | App-only (plaintext XML) | Never for secrets without encryption |
| Room/SQLite database | Medium | App-only (plaintext .db) | Encrypt with SQLCipher for sensitive data |
| External app storage (`getExternalFilesDir()`) | Medium | App-only on API 29+ | Removed on uninstall; was shared on older APIs |
| Shared external storage | Low | Any app with permission | Never for sensitive data |
| ContentProvider | Varies | Depends on export/permissions | Must be properly secured |

## Key Android APIs and Libraries

- **Google Tink** (`com.google.crypto.tink:tink-android`) - Modern cryptographic library with AEAD, streaming AEAD, and deterministic AEAD. Recommended replacement for Jetpack Security.
- **Jetpack DataStore** (`androidx.datastore:datastore-tink`) - Encrypted key-value and proto storage with Tink integration.
- **AndroidKeyStore** - Hardware-backed key storage (TEE/StrongBox). Keys never enter app process memory.
- **SQLCipher** (`net.zetetic:android-database-sqlcipher`) - Transparent SQLite encryption compatible with Room.
- **EncryptedSharedPreferences** - **Deprecated** (April 2025). Migrate to DataStore + Tink.

## OWASP Mobile Top 10 2024 Mapping

- **M9 - Insecure Data Storage:** Directly addressed by MASVS-STORAGE-1
- **M1 - Improper Credential Usage:** Addressed by MASVS-STORAGE-1.3, 1.4, 1.6
- **M8 - Security Misconfiguration:** Addressed by MASVS-STORAGE-1.5, 1.8, MASVS-STORAGE-2.1

---

## MASVS-STORAGE-1

### Control

The app securely stores sensitive data.

### Description

Android apps handle sensitive data from many sources - user input, backend APIs, system services, and other apps - and frequently need to persist it locally. Android provides multiple storage locations with different security properties: internal storage (app-sandboxed), external storage (shared), SharedPreferences, databases, and the AndroidKeyStore. Make sure any sensitive data intentionally stored by the app is properly protected using Android-specific storage mechanisms, encryption APIs, and access controls.

### Android Sub-Requirements

#### MASVS-STORAGE-1.1 - Use Internal Storage for Sensitive Data

The app stores all sensitive data (credentials, tokens, PII, financial data) exclusively in internal storage (`Context.getFilesDir()`, `Context.getCacheDir()`), never on external/shared storage.

**Rationale:** Internal storage is sandboxed per-app by the Linux kernel (UID separation) and is not accessible to other apps without root. External storage (`Environment.getExternalStorageDirectory()`) is accessible to any app with `READ_EXTERNAL_STORAGE` or via USB/ADB.

**Android References:**
- `Context.getFilesDir()` - `/data/data/<package>/files/`
- `Context.getCacheDir()` - `/data/data/<package>/cache/`
- `Context.getNoBackupFilesDir()` - excluded from Auto Backup

#### MASVS-STORAGE-1.2 - Encrypt Sensitive Data at Rest

All sensitive data stored locally is encrypted using authenticated encryption. The app uses `Google Tink` with `AndroidKeysetManager` backed by the AndroidKeyStore, or `Jetpack DataStore` with `AeadSerializer`.

**Rationale:** If a device is rooted, lost, or backed up, plaintext data in internal storage is exposed. Encryption with hardware-backed keys ensures data remains confidential even if the filesystem is compromised.

**Android References:**
- `com.google.crypto.tink:tink-android` - Tink AEAD (AES-256-GCM)
- `AndroidKeysetManager` - key management backed by AndroidKeyStore
- `androidx.datastore:datastore-tink` - encrypted DataStore with `AeadSerializer`
- **Deprecated:** `EncryptedSharedPreferences` (deprecated April 2025, `security-crypto:1.1.0-alpha07`)

#### MASVS-STORAGE-1.3 - Never Store Secrets in SharedPreferences Without Encryption

The app does not store sensitive values (tokens, passwords, API keys, PII) in `SharedPreferences` without encryption. Plain `SharedPreferences` writes XML files to internal storage in cleartext.

**Rationale:** While SharedPreferences files are sandboxed, they are trivially readable on rooted devices, in backups, and via ADB on debuggable builds. Any secret stored in plaintext SharedPreferences should be considered compromised on a rooted device.

#### MASVS-STORAGE-1.4 - Encrypt Databases Containing Sensitive Data

Databases containing sensitive data use `SQLCipher` for Android or equivalent transparent database encryption. If using Room, the app provides an `SupportSQLiteOpenHelper.Factory` backed by SQLCipher.

**Rationale:** SQLite databases in internal storage are plaintext `.db` files. On rooted devices or via backup extraction, their contents are trivially readable with any SQLite client.

**Android References:**
- `net.zetetic:android-database-sqlcipher` - SQLCipher for Android
- `androidx.room:room-runtime` with `SupportSQLiteOpenHelper.Factory` override

#### MASVS-STORAGE-1.5 - Use MODE_PRIVATE for All File Creation

All files created by the app use `Context.MODE_PRIVATE` (the default since API 17). The app never uses `MODE_WORLD_READABLE` or `MODE_WORLD_WRITABLE` (deprecated since API 17, removed in API 24).

**Rationale:** World-readable/writable files break Android's sandbox model and expose data to any app on the device.

#### MASVS-STORAGE-1.6 - Protect AndroidKeyStore Keys with User Authentication

Cryptographic keys protecting sensitive stored data are bound to user authentication via `KeyGenParameterSpec.Builder.setUserAuthenticationRequired(true)` with a defined timeout and authenticator type.

**Rationale:** Without user authentication binding, any process that gains access to the app's UID can use the key. Binding to biometric or device credential ensures the user must be physically present.

**Android References:**
- `KeyGenParameterSpec.Builder.setUserAuthenticationRequired(true)`
- `KeyGenParameterSpec.Builder.setUserAuthenticationParameters(timeout, AUTH_BIOMETRIC_STRONG)`

#### MASVS-STORAGE-1.7 - Prefer StrongBox-Backed Key Storage

When storing cryptographic keys for sensitive data, the app requests StrongBox backing via `KeyGenParameterSpec.Builder.setIsStrongBoxBacked(true)` and gracefully falls back to TEE when StrongBox is unavailable.

**Rationale:** StrongBox provides a dedicated secure processor with its own CPU, memory, and storage, offering stronger tamper resistance than the TEE. It is available on devices with `PackageManager.FEATURE_STRONGBOX_KEYSTORE`.

#### MASVS-STORAGE-1.8 - Exclude Sensitive Data from Auto Backup

The app either disables Auto Backup (`android:allowBackup="false"`) or configures `android:fullBackupContent` / `android:dataExtractionRules` (API 31+) to explicitly exclude files and directories containing sensitive data.

**Rationale:** Android Auto Backup copies app data to Google Drive. If sensitive data is included, it may be accessible via the user's Google account on other devices or by anyone who compromises the account.

**Android References:**
- `android:allowBackup="false"` in `AndroidManifest.xml`
- `android:dataExtractionRules="@xml/data_extraction_rules"` (API 31+)
- `android:fullBackupContent="@xml/backup_rules"` (API 23-30)
- `Context.getNoBackupFilesDir()` - directory excluded from backup by default

#### MASVS-STORAGE-1.9 - Clear Sensitive Data from Memory When No Longer Needed

The app clears sensitive data (passwords, keys, tokens) from memory by overwriting byte arrays and char arrays after use. The app avoids storing secrets in immutable `String` objects where possible.

**Rationale:** Java/Kotlin `String` objects are immutable and remain in the heap until garbage collected. On rooted devices, heap dumps can extract these values. `byte[]` and `char[]` can be explicitly zeroed.

---

## MASVS-STORAGE-2

### Control

The app prevents leakage of sensitive data.

### Description

Sensitive data can be unintentionally exposed through Android platform mechanisms - system logs, clipboard, screenshots, keyboard caches, backups, crash reports, and notification content. These leaks occur as side-effects of using standard APIs and system features. Make sure developers actively prevent these unintentional data exposures using Android-specific mitigations.

### Android Sub-Requirements

#### MASVS-STORAGE-2.1 - No Sensitive Data in System Logs

The app does not write sensitive data (credentials, tokens, PII, session IDs) to system logs via `android.util.Log` or `System.out`/`System.err`. Release builds strip verbose and debug log statements using R8/ProGuard rules.

**Rationale:** On Android versions prior to 4.1, any app could read the system log. On modern Android, logs are accessible via ADB, crash reporting services, and on rooted devices. Log output from production apps should never contain secrets.

**Android References:**
- R8 rule: `-assumenosideeffects class android.util.Log { public static int v(...); public static int d(...); }`
- `Timber` library with a no-op `Tree` for release builds

#### MASVS-STORAGE-2.2 - Prevent Screenshots and Screen Recording of Sensitive Screens

Activities displaying sensitive information (passwords, financial data, OTPs) set `WindowManager.LayoutParams.FLAG_SECURE` to prevent screenshots, screen recording, and display on non-secure outputs (Chromecast, screen mirroring).

**Rationale:** Android captures screenshots for the Recent Apps screen. Without `FLAG_SECURE`, sensitive data appears in the recents thumbnail and can be captured by screen recording apps or remote display.

**Android References:**
- `getWindow().setFlags(FLAG_SECURE, FLAG_SECURE)` in `Activity.onCreate()`
- Android 15+: `WindowManager.addScreenRecordingCallback()` for detection (complementary, not a replacement for FLAG_SECURE)

#### MASVS-STORAGE-2.3 - Disable Keyboard Cache for Sensitive Input Fields

Text input fields accepting sensitive data (passwords, credit card numbers, SSNs) use `android:inputType="textNoSuggestions|textPassword"` or `InputType.TYPE_TEXT_FLAG_NO_SUGGESTIONS` to disable keyboard autocomplete, suggestions, and caching.

**Rationale:** The Android keyboard (IME) caches typed text for autocomplete suggestions. This cache persists on disk and can expose previously typed sensitive values.

**Android References:**
- `android:inputType="textPassword"` - disables suggestions and shows dots
- `android:inputType="textNoSuggestions"` - disables suggestions while showing text
- `EditText.setInputType(InputType.TYPE_TEXT_FLAG_NO_SUGGESTIONS | InputType.TYPE_TEXT_VARIATION_PASSWORD)`

#### MASVS-STORAGE-2.4 - Prevent Sensitive Data in Clipboard

The app prevents copying of sensitive data to the clipboard. Where copy functionality is unavoidable, the app uses `ClipboardManager.setPrimaryClip()` with `ClipDescription.EXTRA_IS_SENSITIVE` (API 33+) to mark content as sensitive, preventing it from appearing in clipboard previews.

**Rationale:** On Android 12 and below, any foreground app can read the clipboard. Android 12+ shows a toast notification on clipboard access. Android 13+ supports auto-clearing and sensitive content flags, but the data is still briefly accessible.

**Android References:**
- `android:longClickable="false"` or custom `ActionMode.Callback` to disable copy on `TextView`/`EditText`
- `ClipDescription.EXTRA_IS_SENSITIVE` (API 33+) - marks clipboard content as sensitive
- `ClipData.newPlainText()` with flag for transient data

#### MASVS-STORAGE-2.5 - Mask Sensitive Data in Notifications

Notifications do not contain sensitive data (OTP codes, account balances, message previews with PII) in their content or title. Where sensitive data is necessary, the app uses `Notification.VISIBILITY_SECRET` or `Notification.VISIBILITY_PRIVATE` with a redacted public version.

**Rationale:** Notifications are visible on the lock screen, in the notification shade, on connected wearables, and in notification history. Sensitive content in notifications is exposed without device unlock.

**Android References:**
- `NotificationCompat.Builder.setVisibility(VISIBILITY_PRIVATE)`
- `NotificationCompat.Builder.setPublicVersion(redactedNotification)`
- `NotificationChannel.setLockscreenVisibility(VISIBILITY_SECRET)`

#### MASVS-STORAGE-2.6 - Disable Autofill for Sensitive Fields Where Inappropriate

Input fields that should not be autofilled (OTP fields, security questions, temporary codes) set `android:importantForAutofill="no"` to opt out of the Android Autofill Framework.

**Rationale:** Autofill services are third-party apps with access to field contents. While legitimate password managers are helpful, autofill of temporary or security-sensitive values introduces unnecessary exposure to the autofill provider.

**Android References:**
- `android:importantForAutofill="no"` - excludes the view from autofill
- `android:importantForAutofill="noExcludeDescendants"` - excludes the view and all children

#### MASVS-STORAGE-2.7 - Prevent Data Leakage via App Backgrounding

The app clears or obscures sensitive UI content when entering the background (`onPause()`/`onStop()`) to prevent exposure in the Recent Apps screen thumbnail.

**Rationale:** Even with `FLAG_SECURE`, some OEMs capture app thumbnails differently. Proactively clearing sensitive views provides defense-in-depth.

#### MASVS-STORAGE-2.8 - Exclude Sensitive Data from Crash Reports and Analytics

The app's crash reporting and analytics SDKs are configured to exclude sensitive data. Custom keys, breadcrumbs, and user attributes do not contain PII, credentials, or session tokens.

**Rationale:** Crash reports are stored on third-party servers (Firebase Crashlytics, Sentry, etc.) and accessible to development teams. Sensitive data in crash reports creates a secondary data exposure surface.

#### MASVS-STORAGE-2.9 - Prevent WebView Cache and Storage Leaks

WebViews that display sensitive content are configured to disable caching (`WebSettings.setCacheMode(LOAD_NO_CACHE)`), clear the cache on navigation away, and disable DOM storage / database storage for sensitive pages.

**Rationale:** WebView caches (HTTP cache, DOM storage, Web SQL databases) persist on the filesystem and can expose sensitive data rendered in web content.

**Android References:**
- `WebSettings.setCacheMode(WebSettings.LOAD_NO_CACHE)`
- `WebView.clearCache(true)` - clears disk and memory cache
- `WebSettings.setDomStorageEnabled(false)` for sensitive contexts

---

## Training-Aligned Requirements

Android-specific requirements for MASVS-STORAGE-1 and MASVS-STORAGE-2.

### MASVS-STORAGE-1: Secure Storage of Sensitive Data

#### STORAGE-ANDROID-1.1: Hardware-Backed Key Storage

The app MUST generate and store cryptographic keys in Android Keystore with hardware backing (TEE or StrongBox). Keys MUST NOT be imported from external sources when hardware generation is available.

**Testable:** Verify via key attestation that `KeyInfo.isInsideSecureHardware()` returns true.

#### STORAGE-ANDROID-1.2: StrongBox Usage for High-Value Keys

For apps processing financial data or sensitive credentials, cryptographic keys SHOULD use StrongBox (physically separate HSM, FIPS 140-2 Level 3) when available on the device.

**Testable:** Verify `setIsStrongBoxBacked(true)` is set in `KeyGenParameterSpec` for high-value key operations. Verify graceful fallback to TEE-backed Keystore when StrongBox is unavailable.

#### STORAGE-ANDROID-1.3: EncryptedSharedPreferences for Local Data

The app MUST use `EncryptedSharedPreferences` (Jetpack Security library) for any sensitive key-value data stored locally. Plain `SharedPreferences` MUST NOT store sensitive data including tokens, credentials, PII, or session identifiers.

**Testable:** Static analysis confirms no sensitive data written to `SharedPreferences`. Dynamic analysis confirms encrypted storage files under `shared_prefs/`.

#### STORAGE-ANDROID-1.4: Encrypted Database Storage

If the app uses local databases for sensitive data, it MUST use encrypted storage such as SQLCipher with Room or equivalent. The database encryption key MUST be stored in Android Keystore, not hardcoded or derived from predictable values.

**Testable:** Inspect database files on disk for plaintext content. Verify encryption key origin traces to Keystore.

#### STORAGE-ANDROID-1.5: External Storage Prohibition

The app MUST NOT write sensitive data to external storage (SD card, public Downloads, or any world-readable location).

**Testable:** Monitor file I/O during app operation. No sensitive data appears in `/sdcard/`, `Environment.getExternalStorageDirectory()`, or `Context.getExternalFilesDir()` paths.

#### STORAGE-ANDROID-1.6: Credential Storage Requirements

Access tokens MUST be stored in memory or encrypted storage with a 5-15 minute lifetime. Refresh tokens MUST be stored in Android Keystore. Credentials MUST NOT be stored in `SharedPreferences`, `NSUserDefaults`, `localStorage`, or unencrypted SQLite databases.

**Testable:** Inspect token storage locations. Verify access token expiry configuration. Confirm refresh tokens are Keystore-protected.

### MASVS-STORAGE-2: Prevention of Sensitive Data Leakage

#### STORAGE-ANDROID-2.1: Backup Exclusion

The app MUST set `android:allowBackup="false"` or use `android:dataExtractionRules` (Android 12+) to explicitly exclude sensitive data from ADB and cloud backups.

**Testable:** Inspect `AndroidManifest.xml` for backup configuration. Attempt ADB backup and verify no sensitive data is extractable.

#### STORAGE-ANDROID-2.2: Log Sanitization

The app MUST NOT log sensitive data (tokens, credentials, PII, cryptographic material) to Android system logs (`Log.d`, `Log.v`, `Log.i`, etc.) in release builds.

**Testable:** Run app with `adb logcat` and exercise all features. No sensitive data appears in log output.

#### STORAGE-ANDROID-2.3: Screenshot and Task Switcher Protection

Activities displaying sensitive data MUST set `FLAG_SECURE` on the window to prevent screenshots and task-switcher thumbnail capture.

**Testable:** Attempt screenshot on sensitive screens. Verify task-switcher shows blank/obscured thumbnail.

#### STORAGE-ANDROID-2.4: Clipboard Protection

The app SHOULD NOT copy sensitive data to the system clipboard. If clipboard use is necessary for sensitive data, the app MUST use `ClipData.newPlainText()` with a content expiration or instruct users about the risk.

**Testable:** Monitor clipboard contents during app use. Verify no persistent sensitive data in clipboard.

#### STORAGE-ANDROID-2.5: Keyboard Cache Prevention

For input fields accepting sensitive data (passwords, credit cards, OTPs), the app MUST disable keyboard suggestions and caching using `android:inputType="textNoSuggestions|textPassword"` or equivalent.

**Testable:** Inspect input field XML attributes. Verify keyboard dictionaries do not cache sensitive input.

#### STORAGE-ANDROID-2.6: WebView Cache and Storage Cleanup

If the app uses WebViews, it MUST clear WebView cache, cookies, and local storage when the user logs out or the session ends.

**Testable:** After logout, inspect WebView data directories for residual sensitive data.

#### STORAGE-ANDROID-2.7: OTP Protection

On Android 15+, the app SHOULD use OTP protection features that hide OTPs from notification previews.

**Testable:** Send OTP to device. Verify notification content does not expose the OTP code on lock screen.
