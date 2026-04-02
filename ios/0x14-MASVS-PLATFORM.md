# MASVS-PLATFORM: Platform Interaction

## Overview

iOS IPC mechanisms include Universal Links, custom URL schemes, App Groups, Keychain Access Groups, UIPasteboard, App Extensions, and WKWebView. Universal Links verify domain ownership via an Apple App Site Association (AASA) file; custom URL schemes do not. WKWebView runs web content in a separate process but still requires careful configuration to prevent JavaScript bridge abuse, file access, and XSS. Make sure all platform interactions are secured against iOS-specific attack patterns.

## iOS IPC Attack Surface

| Mechanism | Attack Vector | Key Risk |
|---|---|---|
| Custom URL Scheme | Scheme hijacking, parameter injection | Any app can register the same scheme |
| Universal Links | AASA misconfiguration | Link handled by wrong app if not verified |
| App Groups (shared container) | Data exposure to other group apps | Sensitive data in shared container |
| Keychain Access Groups | Cross-app credential access | Must be same team ID |
| UIPasteboard | Clipboard sniffing (pre-iOS 16) | Shared across all apps |
| WKWebView | JS bridge abuse, file access, XSS | Bridge exposes native methods to web |
| App Extensions | Shared container data leakage | Extension processes data outside main app |
| Handoff / NSUserActivity | Activity data interception | Activities may contain sensitive URLs |

## iOS Version Platform Protections

| iOS Version | Protection |
|---|---|
| 12 | UIWebView deprecated |
| 14 | App Attest, App Clips, local network permission |
| 16 | Paste permission prompt |
| 17 | Link Tracking Protection |
| 18 | Reduced Contact Access, Locked Apps |
| 26 | Accessory restriction controls |

## Controls Summary

| Control | Title |
|---|---|
| MASVS-PLATFORM-1 | The app uses IPC mechanisms securely |
| MASVS-PLATFORM-2 | The app uses WebViews securely |
| MASVS-PLATFORM-3 | The app uses the user interface securely |

## OWASP Mobile Top 10 2024 Mapping

- **M4 - Insufficient Input/Output Validation:** Addressed by MASVS-PLATFORM-1.2, 2.2, 2.4
- **M8 - Security Misconfiguration:** Addressed by MASVS-PLATFORM-1.1, 2.1, 2.3

---

## MASVS-PLATFORM-1

### Control

The app uses IPC mechanisms securely.

### Description

iOS IPC mechanisms can expose the app to attacks if misconfigured. Custom URL schemes have no ownership verification. Universal Links prove domain ownership but still receive untrusted input. Make sure all IPC entry points validate input and use verified mechanisms.

### iOS Sub-Requirements

#### MASVS-PLATFORM-1.1 - Use Universal Links, Not Custom URL Schemes

Security-sensitive entry points (auth callbacks, payment flows, sensitive navigation) use Universal Links with a valid AASA file, not custom URL schemes.

**Rationale:** Any app can register the same custom URL scheme. There is no ownership verification. Universal Links verify domain ownership via AASA.

**iOS References:**
- Apple App Site Association (AASA) file hosted at `https://domain/.well-known/apple-app-site-association`
- `NSUserActivity` with `activityType == NSUserActivityTypeBrowsingWeb` for Universal Link handling

#### MASVS-PLATFORM-1.2 - Validate All Deep Link Parameters

All parameters from Universal Links and URL schemes are validated against expected types, ranges, and patterns. Deep link parameters are never passed directly to sensitive operations without validation.

**Rationale:** Deep links receive untrusted input from any app or website.

#### MASVS-PLATFORM-1.3 - Never Auto-Execute Sensitive Actions from Deep Links

Deep links do not automatically execute sensitive actions (payments, account changes, data deletion). The user must confirm before any sensitive action triggered by a deep link.

**Rationale:** An attacker can craft links that trigger unintended actions.

#### MASVS-PLATFORM-1.4 - Minimize App Group Shared Container Data

Sensitive data in App Group shared containers is minimized. The shared container does not contain credentials, tokens, or encryption keys. If sensitive data must be shared, it is encrypted with a key from the Keychain.

**Rationale:** All apps and extensions in the group can access the shared container.

**iOS References:**
- `FileManager.containerURL(forSecurityApplicationGroupIdentifier:)` for shared container access
- Keychain for storing encryption keys used to protect shared data

#### MASVS-PLATFORM-1.5 - Restrict Keychain Access Groups

Keychain Access Groups are limited to the same team ID. The `kSecAttrAccessGroup` attribute is explicitly set to limit which apps can access shared Keychain items.

**Rationale:** Overly permissive access groups allow unintended cross-app credential sharing.

**iOS References:**
- `kSecAttrAccessGroup` in Keychain queries
- Keychain Access Groups entitlement in the app's provisioning profile

#### MASVS-PLATFORM-1.6 - Secure Pasteboard Usage

The app avoids copying sensitive data (credentials, tokens, financial data) to UIPasteboard. If copying sensitive data is necessary, use `localOnly` and `expirationDate` options on `UIPasteboard.general.setItems(_:options:)`.

**Rationale:** Before iOS 16, any foreground app could silently read the pasteboard. iOS 16+ prompts the user, but the data is still accessible if the user approves.

**iOS References:**
- `UIPasteboard.general.setItems(_:options:)` with `localOnly` and `expirationDate`

#### MASVS-PLATFORM-1.7 - Validate App Extension Input

App Extensions validate all input received from the host app via `NSExtensionContext`. Extensions minimize data stored in the shared container and clean up temporary data after processing.

**Rationale:** Extensions process data outside the main app's context. The host app is untrusted.

**iOS References:**
- `NSExtensionContext.inputItems` for receiving data from the host app

#### MASVS-PLATFORM-1.8 - Secure Handoff / NSUserActivity

NSUserActivity objects do not contain sensitive data (credentials, tokens, PII). Activity continuation validates the source. The `userInfo` dictionary does not store sensitive state.

**Rationale:** Handoff activities transfer between devices on the same iCloud account and may contain URLs or state data.

**iOS References:**
- `NSUserActivity.userInfo` for activity state
- `application(_:continue:restorationHandler:)` for handling continued activities

---

## MASVS-PLATFORM-2

### Control

The app uses WebViews securely.

### Description

WKWebView runs web content in a separate process, providing isolation from the main app process. However, JavaScript bridges, file access, and navigation to untrusted content remain risks. Make sure WebViews are configured with least privilege.

### iOS Sub-Requirements

#### MASVS-PLATFORM-2.1 - Use WKWebView Exclusively

The app does not use UIWebView (deprecated iOS 12, rejected by App Store). All web content uses WKWebView.

**Rationale:** UIWebView runs in the app's process with no isolation. The App Store rejects new submissions using UIWebView.

**iOS References:**
- `WKWebView` (WebKit framework)
- UIWebView deprecated since iOS 12

#### MASVS-PLATFORM-2.2 - Validate WKScriptMessageHandler Input

All messages received via `WKScriptMessageHandler` (JavaScript-to-native bridge) validate their content against expected types and values. The handler does not expose sensitive operations without input validation and authorization checks.

**Rationale:** Any JavaScript running in the WebView can send messages to native code via `window.webkit.messageHandlers.<name>.postMessage()`.

**iOS References:**
- `WKScriptMessageHandler` protocol
- `userContentController(_:didReceive:)` for receiving messages

#### MASVS-PLATFORM-2.3 - Disable JavaScript for Untrusted Content

WebViews loading untrusted content disable JavaScript via `WKWebViewConfiguration.defaultWebpagePreferences.allowsContentJavaScript = false`.

**Rationale:** JavaScript in untrusted content enables XSS and bridge exploitation.

**iOS References:**
- `WKWebpagePreferences.allowsContentJavaScript`

#### MASVS-PLATFORM-2.4 - Implement WKNavigationDelegate URL Validation

The app implements `webView(_:decidePolicyFor:decisionHandler:)` to validate navigation URLs against a domain allowlist. Navigation to `javascript:`, `data:`, and untrusted domains is blocked by calling `decisionHandler(.cancel)`.

**Rationale:** Without validation, attackers can redirect the WebView to malicious content.

**iOS References:**
- `WKNavigationDelegate` protocol
- `webView(_:decidePolicyFor:decisionHandler:)` for navigation policy decisions

#### MASVS-PLATFORM-2.5 - Handle SSL Errors by Canceling Navigation

The `webView(_:didReceive:completionHandler:)` implementation cancels navigation for server trust challenges that fail validation. The app never calls `completionHandler(.performDefaultHandling, nil)` or `completionHandler(.useCredential, credential)` for untrusted certificates.

**Rationale:** Proceeding through SSL errors disables certificate validation for the WebView.

**iOS References:**
- `WKNavigationDelegate.webView(_:didReceive:completionHandler:)` for authentication challenges
- `URLAuthenticationChallenge` with `SecTrust` evaluation

#### MASVS-PLATFORM-2.6 - Clear WebView Data After Sensitive Sessions

After sensitive content, the app clears data using `WKWebsiteDataStore.default().removeData(ofTypes:modifiedSince:completionHandler:)`. The data types cleared include cookies, local storage, caches, and session storage.

**Rationale:** WebView caches persist on the filesystem and can be extracted from a jailbroken device.

**iOS References:**
- `WKWebsiteDataStore.default().removeData(ofTypes:modifiedSince:completionHandler:)`
- `WKWebsiteDataRecord` types: `WKWebsiteDataTypeCookies`, `WKWebsiteDataTypeLocalStorage`, `WKWebsiteDataTypeDiskCache`

#### MASVS-PLATFORM-2.7 - Use WKContentRuleList for Content Blocking

For WebViews loading third-party content, the app uses `WKContentRuleList` to block unwanted resources (trackers, ads, known malicious domains).

**Rationale:** Content rules provide declarative, efficient content blocking without requiring JavaScript injection.

**iOS References:**
- `WKContentRuleListStore.default().compileContentRuleList(forIdentifier:encodedContentRuleList:completionHandler:)`
- `WKWebViewConfiguration.userContentController.add(_:)` for adding compiled rule lists

---

## MASVS-PLATFORM-3

### Control

The app uses the user interface securely.

### Description

Sensitive UI data can be exposed through screenshots, screen recording, accessibility abuse, and social engineering. iOS provides mechanisms to protect displayed content. Make sure the app's UI properly protects displayed sensitive information using iOS-specific mitigations.

### iOS Sub-Requirements

#### MASVS-PLATFORM-3.1 - Protect Sensitive Screens from Screenshot Capture

The app overlays or clears sensitive content in `applicationWillResignActive` / `sceneWillResignActive` and restores it in `applicationDidBecomeActive` / `sceneDidBecomeActive`. Alternatively, the app uses a hidden `UITextField` with `secureTextEntry` to prevent screenshots of sensitive views.

**Rationale:** iOS captures a screenshot for the app switcher when the app backgrounds. Screen recording and screenshots also capture visible content.

**iOS References:**
- `UIApplicationDelegate.applicationWillResignActive(_:)` / `UISceneDelegate.sceneWillResignActive(_:)`
- `UITextField.isSecureTextEntry` trick for screenshot-proof views

#### MASVS-PLATFORM-3.2 - Mask Sensitive Input

Password fields use `secureTextEntry = true`. Other sensitive fields (credit cards, SSNs) are partially masked with reveal toggles. Auto-correction and predictive text are disabled for sensitive fields (`autocorrectionType = .no`, `spellCheckingType = .no`).

**Rationale:** Displayed sensitive data is visible to shoulder surfers, screen recordings, and accessibility services. Auto-correction caches sensitive input.

**iOS References:**
- `UITextField.isSecureTextEntry`
- `UITextField.autocorrectionType`
- `UITextField.spellCheckingType`

#### MASVS-PLATFORM-3.3 - Provide Clear Permission Request Context

Each permission request includes a meaningful `NSUsageDescription` string in `Info.plist` explaining why the permission is needed. Permissions are requested at point of use, not on launch.

**Rationale:** Generic or absent descriptions confuse users and may cause App Store rejection.

**iOS References:**
- `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSLocationWhenInUseUsageDescription`, etc. in `Info.plist`

#### MASVS-PLATFORM-3.4 - iOS 14+: Selected Photos Access

The app uses `PHPickerViewController` or limited photo library access instead of requesting full `READ_MEDIA` access when only specific user-selected photos are needed.

**Rationale:** Full photo library access exposes all user photos. `PHPickerViewController` runs out-of-process and does not require photo library permission.

**iOS References:**
- `PHPickerViewController` (PhotosUI framework, iOS 14+)
- `PHAuthorizationStatus.limited` for limited photo library access

#### MASVS-PLATFORM-3.5 - iOS 18: Reduced Contact Access

When only specific contacts are needed, the app supports limited contact access where the user selects which contacts to share.

**Rationale:** Full address book access exposes all user contacts. Limited contact access restricts the app to only user-selected contacts.

**iOS References:**
- `CNContactPickerViewController` for user-selected contact access
- iOS 18 limited contact authorization

#### MASVS-PLATFORM-3.6 - Validate QR Code and NFC Input

Decoded QR code and NFC data is treated as untrusted input. The app displays decoded URLs to the user before navigating, validates against an allowlist, and never auto-executes actions from scanned data.

**Rationale:** QR codes bypass email filters and can be placed anywhere. NFC tags can be cloned or maliciously written.

**iOS References:**
- `AVCaptureMetadataOutput` for QR code scanning
- `CoreNFC` framework for NFC tag reading
- `SFSafariViewController` or user confirmation before opening scanned URLs

#### MASVS-PLATFORM-3.7 - iOS 18: Respect Locked and Hidden Apps

The app handles the case where it may be locked behind Face ID/Touch ID. Sensitive notifications and Siri suggestions respect the app's locked status. The app does not expose sensitive content in notification previews when locked.

**Rationale:** iOS 18 allows users to lock apps behind biometric authentication. Apps must not leak sensitive data through notifications or Siri suggestions when locked.

**iOS References:**
- `UNNotificationContent.hiddenPreviewsBodyPlaceholder` for locked notification previews
- Locked and Hidden Apps (iOS 18)

---

## Training-Aligned Requirements

iOS-specific requirements for MASVS-PLATFORM-1, MASVS-PLATFORM-2, and MASVS-PLATFORM-3.

### MASVS-PLATFORM-1: Secure IPC Mechanisms

#### PLATFORM-IOS-1.1: Universal Links

Security-sensitive deep links MUST use Universal Links with a verified AASA file. Custom URL schemes MUST NOT be used for authentication callbacks, payment flows, or sensitive navigation.

**Testable:** Verify AASA file is hosted and valid. Test that custom scheme is not used for sensitive entry points. Verify Universal Link handling in `application(_:continue:restorationHandler:)`.

#### PLATFORM-IOS-1.2: Deep Link Validation

All deep link parameters MUST be validated as untrusted input. Parameters MUST be type-checked and range-validated before use in any operation.

**Testable:** Send crafted Universal Links and URL scheme links with malicious parameters. Verify validation rejects unexpected input. Verify sensitive actions require user confirmation.

### MASVS-PLATFORM-2: Secure WebView Usage

#### PLATFORM-IOS-2.1: WKWebView Only

UIWebView MUST NOT be used. All web content MUST use WKWebView.

**Testable:** Scan binary for UIWebView class references using `nm` or `otool`. Scan source for UIWebView imports. Verify App Store submission does not trigger UIWebView rejection.

#### PLATFORM-IOS-2.2: JS Bridge Validation

`WKScriptMessageHandler` implementations MUST validate all input received from JavaScript. Handlers MUST NOT expose sensitive operations without input validation.

**Testable:** Send crafted messages from injected JavaScript via `window.webkit.messageHandlers`. Verify handlers reject unexpected message types and values.

#### PLATFORM-IOS-2.3: URL Allowlisting

WebView navigation MUST be restricted to allowlisted domains via `WKNavigationDelegate`. Navigation to `javascript:`, `data:`, and non-allowlisted domains MUST be blocked.

**Testable:** Attempt navigation to non-allowlisted URL. Verify `decisionHandler(.cancel)` is called. Verify `javascript:` and `data:` schemes are blocked.

### MASVS-PLATFORM-3: Secure UI

#### PLATFORM-IOS-3.1: Screenshot Protection

Sensitive screens MUST obscure content when the app backgrounds. The app MUST implement `sceneWillResignActive` / `applicationWillResignActive` to overlay or clear sensitive views.

**Testable:** Background the app on a sensitive screen. Check the app switcher thumbnail for sensitive data. Verify content is obscured.

#### PLATFORM-IOS-3.2: Permission Strings

All permission requests MUST have meaningful `NSUsageDescription` strings in `Info.plist`. Permissions MUST be requested at point of use, not at launch.

**Testable:** Inspect `Info.plist` for all declared `NSUsageDescription` keys. Verify descriptions are specific and meaningful. Launch app and verify no permission prompts appear before the relevant feature is accessed.

#### PLATFORM-IOS-3.3: QR/NFC Validation

QR code and NFC data MUST be validated as untrusted input. Decoded URLs MUST be displayed to the user before navigation. Auto-execution of scanned actions MUST NOT occur.

**Testable:** Scan crafted QR codes with malicious URLs. Verify URL is displayed for user confirmation. Verify non-allowlisted URLs are blocked or warned.
