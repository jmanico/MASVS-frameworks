# MASVS-PLATFORM: Platform Interaction

## Overview

Android's component model (Activities, Services, Broadcast Receivers, and Content Providers), combined with the Intent system, WebViews, and deep links, creates a rich but complex attack surface. Every exported component is an entry point accessible to any app on the device. WebViews bridge web content to native code. Deep links expose the app to the internet. Make sure all platform interactions are secured against the Android-specific attack patterns that exploit these mechanisms.

## Android IPC Attack Surface

| Component | Attack Vector | Key Risk |
|---|---|---|
| Exported Activity | Intent injection, task hijacking (StrandHogg) | Unauthorized access, credential phishing |
| Content Provider | SQL injection, path traversal, data leakage | Data theft, arbitrary file read |
| Broadcast Receiver | Broadcast theft, spoofed broadcasts | Data interception, unauthorized actions |
| Service / Bound Service | Unauthorized service invocation | Privilege escalation, data access |
| PendingIntent | Mutable intent hijacking | Redirect operations to malicious targets |
| Deep Link | Parameter injection, unvalidated navigation | Arbitrary activity launch, XSS |
| WebView | JavaScript bridge abuse, file access, XSS | Native code execution, data theft |
| App Links | Link hijacking (if not verified) | Credential interception |

## Android Version-Specific Protections

| API Level | Protection |
|---|---|
| 17 | `MODE_WORLD_READABLE`/`WRITABLE` deprecated |
| 24 | User CAs not trusted; `FileProvider` recommended |
| 28 | Cleartext blocked; Protected Confirmation |
| 30 | Package visibility restrictions |
| 31 | Explicit `exported` required; `FLAG_IMMUTABLE` for PendingIntent |
| 33 | `RECEIVER_NOT_EXPORTED` flag |
| 35 | Safer intent matching; screen recording detection |
| 36 | Safer Intents v2; intent redirection hardening; local network permission |

## Controls Summary

| Control | Title |
|---|---|
| MASVS-PLATFORM-1 | The app uses IPC mechanisms securely |
| MASVS-PLATFORM-2 | The app uses WebViews securely |
| MASVS-PLATFORM-3 | The app uses the user interface securely |

## OWASP Mobile Top 10 2024 Mapping

- **M4 - Insufficient Input/Output Validation:** Addressed by MASVS-PLATFORM-1.4, 1.5, 1.7, 2.2, 2.4
- **M8 - Security Misconfiguration:** Addressed by MASVS-PLATFORM-1.1, 1.2, 2.1, 2.3, 2.5
- **M3 - Insecure Authentication/Authorization:** Addressed by MASVS-PLATFORM-3.1, 3.5

---

## MASVS-PLATFORM-1

### Control

The app uses IPC mechanisms securely.

### Description

Android's Inter-Process Communication (IPC) mechanisms (Intents, Content Providers, Broadcast Receivers, and Services) are fundamental to the platform but also a major attack surface. Exported components are accessible to any app on the device. Implicit intents can be intercepted. Content Providers can leak data or be exploited via SQL injection and path traversal. Make sure all IPC interactions on Android are secured against common attack patterns.

### Android Sub-Requirements

#### MASVS-PLATFORM-1.1 - Minimize Exported Components

All components (Activities, Services, Broadcast Receivers, Content Providers) are set to `android:exported="false"` unless they explicitly need to receive intents from other apps. Every `android:exported="true"` component has documented justification and appropriate permission protection.

**Rationale:** Since Android 12 (API 31), components with intent-filters must explicitly declare `android:exported`. Every exported component is an entry point that any app on the device can invoke.

**Android References:**
- `android:exported="false"` in `AndroidManifest.xml` (default for components without intent-filters since API 31)

#### MASVS-PLATFORM-1.2 - Protect Exported Components with Permissions

Exported components that handle sensitive operations or data are protected with custom signature-level permissions (`android:protectionLevel="signature"`) or system permissions. The `android:permission` attribute is set on the component declaration.

**Android References:**
- XML example with custom permission and protected activity

#### MASVS-PLATFORM-1.3 - Use Explicit Intents for Sensitive Communication

The app uses explicit intents (specifying the target component by `ComponentName` or class) for all sensitive IPC. Implicit intents are only used for non-sensitive, user-facing actions (e.g., opening a browser, sharing non-sensitive content).

**Rationale:** Implicit intents can be intercepted by any app that registers a matching intent-filter. On Android 15+, explicit intents must match the target's intent-filter specs, and intents without an action cannot match filters.

#### MASVS-PLATFORM-1.4 - Validate All Intent Data

The app validates and sanitizes all data received from intents, including extras, data URIs, actions, categories, and clipData. The app does not blindly trust intent data, even from seemingly trusted sources. Specific validations: URI scheme, host, and path are validated against an allowlist; Intent extras are type-checked and range-validated; The app does not call `Intent.parseUri()` or `Intent.getParcelableExtra()` on untrusted data without validation; Deep links are validated against expected patterns before navigating.

#### MASVS-PLATFORM-1.5 - Prevent Intent Redirection

The app does not forward or redirect user-controlled Intent data to `startActivity()`, `startService()`, `sendBroadcast()`, or `setResult()` without validation. Intent redirection allows an attacker to access unexported components through an exported component that acts as a proxy.

**Rationale:** Intent redirection is a critical vulnerability class on Android. An attacker crafts an intent containing a nested intent (in extras), which the vulnerable app then launches on the attacker's behalf, bypassing export restrictions.

**Android References:**
- Android 16+: by-default protection against intent redirection exploits
- `IntentSanitizer` (Jetpack) for validating received intents

#### MASVS-PLATFORM-1.6 - Use FLAG_IMMUTABLE for PendingIntents

All `PendingIntent` objects are created with `PendingIntent.FLAG_IMMUTABLE` unless the app requires the intent to be modified by the receiving app (in which case `FLAG_MUTABLE` is used with explicit component, action, and package set on the inner intent).

**Rationale:** Mutable PendingIntents with implicit inner intents can be hijacked. A malicious app can fill in the unspecified fields to redirect the PendingIntent to its own component. `FLAG_IMMUTABLE` is required for apps targeting API 31+.

**Android References:**
- `PendingIntent.getActivity(context, requestCode, intent, FLAG_IMMUTABLE)`
- `PendingIntent.FLAG_IMMUTABLE` (required targeting API 31+)

#### MASVS-PLATFORM-1.7 - Secure Content Providers

Content Providers that are exported: Use parameterized queries (via `ContentResolver` with selection args) to prevent SQL injection; Validate file paths in `openFile()` to prevent path traversal (`../../` attacks); Set appropriate `android:readPermission` and `android:writePermission`; Use `android:grantUriPermissions="true"` with `FLAG_GRANT_READ_URI_PERMISSION` for temporary, scoped access instead of broad export; Use `FileProvider` for file sharing instead of custom Content Provider implementations.

**Android References:**
- `FileProvider` - secure file sharing via content URIs
- `ContentProvider.query()` with `selectionArgs` parameter for parameterized queries

#### MASVS-PLATFORM-1.8 - Secure Broadcast Receivers

Broadcast Receivers that handle sensitive data are registered with `ContextCompat.registerReceiver(context, receiver, filter, RECEIVER_NOT_EXPORTED)` (API 33+) or protected with `android:permission`; The app uses explicit broadcasts (`Intent.setComponent()` or `Intent.setPackage()`) when sending sensitive data; Ordered broadcasts containing sensitive data specify a permission that receivers must hold; The app does not send sensitive data via implicit broadcasts.

#### MASVS-PLATFORM-1.9 - Secure Bound Services

AIDL-based bound services verify the caller's identity using `Binder.getCallingUid()` and `PackageManager.getPackagesForUid()` before processing sensitive requests; Services protected with `android:permission` use signature-level permissions for same-developer communication; The service validates all data received through AIDL/Messenger interfaces.

#### MASVS-PLATFORM-1.10 - Secure Deep Links and App Links

Deep link handlers validate the incoming URI scheme, host, path, and query parameters against an allowlist; Android App Links (verified `https://` links with `android:autoVerify="true"`) are preferred over custom scheme deep links for security-sensitive entry points; The app does not pass deep link parameters directly to sensitive operations without validation; The `.well-known/assetlinks.json` file is properly configured on the web server.

**Android References:**
- `<intent-filter android:autoVerify="true">` with `ACTION_VIEW` and `https` data scheme
- Digital Asset Links (`.well-known/assetlinks.json`) for domain verification

---

## MASVS-PLATFORM-2

### Control

The app uses WebViews securely.

### Description

Android WebViews (`android.webkit.WebView`) embed web content within native apps and bridge JavaScript to native code. Misconfigured WebViews are a major attack surface. They can expose native functionality to malicious web content, leak sensitive data through caches and cookies, and enable cross-site scripting within the app's context. Make sure WebViews are configured with the principle of least privilege and that JavaScript bridges are properly secured.

### Android Sub-Requirements

#### MASVS-PLATFORM-2.1 - Disable JavaScript Unless Required

WebViews have JavaScript disabled by default (`WebSettings.setJavaScriptEnabled(false)`). JavaScript is only enabled when the WebView must render interactive web content that requires it, and only for trusted content.

**Rationale:** JavaScript enables dynamic execution of code within the WebView. If the WebView loads untrusted content (user-generated, third-party, or attacker-controlled), JavaScript can be used for XSS, data exfiltration, and native bridge exploitation.

#### MASVS-PLATFORM-2.2 - Secure JavaScript Interface Bridges

If `addJavascriptInterface()` is used: The interface object exposes only the minimum necessary methods; All methods annotated with `@JavascriptInterface` validate their inputs; The interface does not expose access to sensitive data, file system operations, or native capabilities that could be abused; The WebView only loads content from trusted, verified origins.

**Rationale:** `addJavascriptInterface()` exposes native Java/Kotlin methods to JavaScript running in the WebView. On API 16 and below, all public methods (including inherited ones from `Object`) were accessible, which was a critical RCE vulnerability. On API 17+, only `@JavascriptInterface`-annotated methods are exposed, but they remain callable by any JavaScript running in the WebView.

#### MASVS-PLATFORM-2.3 - Disable File Access in WebViews

WebViews are configured with: `WebSettings.setAllowFileAccess(false)`, `WebSettings.setAllowFileAccessFromFileURLs(false)`, `WebSettings.setAllowUniversalAccessFromFileURLs(false)`, `WebSettings.setAllowContentAccess(false)` - disables `content://` scheme access (unless specifically needed).

**Rationale:** File access from WebViews enables local file theft. A malicious page loaded in the WebView can read files from the app's internal storage (credentials, databases) if file access is enabled.

#### MASVS-PLATFORM-2.4 - Validate URLs in shouldOverrideUrlLoading

The app implements `WebViewClient.shouldOverrideUrlLoading()` and validates all navigation URLs against an allowlist of trusted schemes and domains. Navigation to `javascript:`, `data:`, `file:`, and untrusted `https://` domains is blocked.

**Rationale:** Without URL validation, an attacker can navigate the WebView to malicious content via crafted links, redirects, or JavaScript. Scheme validation prevents `javascript:` URI exploitation and `file://` access.

#### MASVS-PLATFORM-2.5 - Handle SSL Errors Correctly in WebViews

The app's `WebViewClient.onReceivedSslError()` implementation calls `handler.cancel()` for all SSL errors. It never calls `handler.proceed()` in production builds. The user may be shown an error message, but the connection is always terminated.

**Rationale:** Calling `handler.proceed()` on SSL errors disables certificate validation for the WebView, enabling MITM attacks on all content loaded in that WebView. Google Play rejects apps that call `handler.proceed()`.

#### MASVS-PLATFORM-2.6 - Clear WebView Data After Sensitive Sessions

After loading sensitive content, the app clears WebView state: `WebView.clearCache(true)`, `WebView.clearHistory()`, `CookieManager.getInstance().removeAllCookies(null)`, `WebStorage.getInstance().deleteAllData()`.

**Rationale:** WebView caches, cookies, and local storage persist on the filesystem and can be extracted from the device.

#### MASVS-PLATFORM-2.7 - Use WebView Safe Browsing

The app enables Safe Browsing for WebViews via manifest metadata.

**Android References:**
- Available since API 26; enabled by default since API 28 but can be explicitly declared for clarity
- `WebView.startSafeBrowsing(context, callback)` for async initialization

```xml
<meta-data android:name="android.webkit.WebView.EnableSafeBrowsing"
    android:value="true" />
```

---

## MASVS-PLATFORM-3

### Control

The app uses the user interface securely.

### Description

Sensitive data displayed in the Android UI (passwords, credit card numbers, OTP codes, financial balances) can be exposed through platform mechanisms such as screenshots for the Recent Apps screen, screen overlays (tapjacking), shoulder surfing, screen sharing, and accessibility services. Make sure the app's UI properly protects displayed sensitive information using Android-specific mitigations.

### Android Sub-Requirements

#### MASVS-PLATFORM-3.1 - Prevent Tapjacking / Overlay Attacks

Views that perform security-sensitive actions (login, payment authorization, permission grants) are protected against overlay attacks: Set `android:filterTouchesWhenObscured="true"` on sensitive views in layout XML; Override `onFilterTouchEventForSecurity()` to check `MotionEvent.FLAG_WINDOW_IS_OBSCURED` and `FLAG_WINDOW_IS_PARTIALLY_OBSCURED`; Reject touch events when the view is obscured by another window.

**Rationale:** Tapjacking (overlay attacks) involve a malicious app drawing a transparent or misleading overlay on top of the legitimate app's UI, tricking the user into tapping sensitive controls. Research in 2025 ("TapTrap") demonstrated animation-driven tapjacking on Android 15+ without overlay permission.

**Android References:**
- `android:filterTouchesWhenObscured="true"` in layout XML
- `View.onFilterTouchEventForSecurity(MotionEvent)` - return false to reject obscured touches
- `MotionEvent.FLAG_WINDOW_IS_OBSCURED` and `FLAG_WINDOW_IS_PARTIALLY_OBSCURED`

#### MASVS-PLATFORM-3.2 - Use FLAG_SECURE for Sensitive Screens

Activities displaying sensitive information set `FLAG_SECURE` to prevent screenshots, screen recording, display on non-secure outputs (casting, mirroring), and thumbnail capture for the Recent Apps screen. (Cross-reference: MASVS-STORAGE-2.2).

**Android References:**
- `getWindow().setFlags(WindowManager.LayoutParams.FLAG_SECURE, WindowManager.LayoutParams.FLAG_SECURE)`

#### MASVS-PLATFORM-3.3 - Detect Screen Recording (Android 15+)

On Android 15+, apps handling sensitive data register a `WindowManager.addScreenRecordingCallback()` to detect when the screen is being recorded and take appropriate action (obscuring sensitive content, pausing sensitive operations, or notifying the user).

**Rationale:** `FLAG_SECURE` prevents screenshots and recording. The screen recording callback provides additional awareness. The app can adjust behavior (e.g., hide account balances) when recording is active, even for non-FLAG_SECURE screens.

#### MASVS-PLATFORM-3.4 - Mask Sensitive Data in the UI

Sensitive data displayed in the UI is masked or partially redacted by default: Passwords use `android:inputType="textPassword"` (shows dots); Credit card numbers show only the last 4 digits; Account numbers and SSNs are partially masked; A "reveal" toggle is provided where the user needs to see the full value.

#### MASVS-PLATFORM-3.5 - Prevent Task Hijacking

The app mitigates StrandHogg/task hijacking attacks: Set `android:taskAffinity=""` (empty string) on sensitive Activities; Set `android:launchMode="singleTask"` or `android:launchMode="singleInstance"` for the main Activity; Consider `android:allowTaskReparenting="false"` to prevent Activities from being moved between tasks.

**Rationale:** Task hijacking (StrandHogg) allows a malicious app to place its Activity on top of the legitimate app's task stack by manipulating task affinity. The user believes they are interacting with the legitimate app but is actually providing credentials to the malicious app.

#### MASVS-PLATFORM-3.6 - Secure Accessibility Service Interaction

The app does not expose sensitive data unnecessarily to accessibility services: Sensitive views set `android:importantForAccessibility="no"` where accessibility is not needed; Password fields use `android:inputType="textPassword"` which automatically masks content from accessibility services; The app does not broadcast sensitive data through `AccessibilityEvent` custom content descriptions.

**Rationale:** Accessibility services have broad access to UI content and events. While legitimate accessibility services are essential, malicious apps can abuse accessibility permissions to read all displayed text, capture credentials, and perform actions on behalf of the user.

---

## Training-Aligned Requirements

Android-specific requirements for MASVS-PLATFORM-1, MASVS-PLATFORM-2, and MASVS-PLATFORM-3.

### MASVS-PLATFORM-1: Secure IPC Mechanisms

#### PLATFORM-ANDROID-1.1: Explicit Component Export

All components (Activities, Services, BroadcastReceivers, ContentProviders) MUST explicitly declare `android:exported="true"` or `android:exported="false"` in the manifest. This is mandatory since Android 12 (API 31).

**Testable:** Inspect `AndroidManifest.xml`. Verify all components have explicit `exported` attribute. Build fails if omitted on API 31+.

#### PLATFORM-ANDROID-1.2: Minimal Component Exposure

Only components that must be accessible to other apps SHOULD be exported. Sensitive exported components MUST be protected with `signature`-level custom permissions.

**Testable:** List all exported components. Verify each has a documented reason for export. Verify sensitive exported components use signature-level permissions.

#### PLATFORM-ANDROID-1.3: Immutable PendingIntents

All `PendingIntent` objects MUST use `FLAG_IMMUTABLE` (default since Android 12) unless mutability is explicitly required for the use case. Mutable PendingIntents MUST NOT be sent to untrusted components.

**Testable:** Static analysis for `PendingIntent.get*()` calls. Verify `FLAG_IMMUTABLE` is set. Verify no mutable PendingIntents exposed to untrusted components.

#### PLATFORM-ANDROID-1.4: Intent Redirection Protection

The app MUST NOT forward untrusted Intents to internal components without validation. On Android 16+, the app SHOULD enable Intent Redirection Protection via StrictMode.

**Testable:** Identify Intent forwarding patterns. Verify input validation on forwarded Intents. On Android 16+, verify StrictMode intent redirection detection is enabled.

#### PLATFORM-ANDROID-1.5: Package Visibility Scoping

On Android 11+ (API 30), the app MUST use the `<queries>` element in the manifest to declare which packages it needs to interact with, rather than requesting broad package visibility via `QUERY_ALL_PACKAGES`.

**Testable:** Inspect manifest for `<queries>` usage. Verify `QUERY_ALL_PACKAGES` is not declared unless justified and documented.

### MASVS-PLATFORM-2: Secure WebView Usage

#### PLATFORM-ANDROID-2.1: Disabled File and Content Access

WebViews MUST disable file and content access: `setAllowFileAccess(false)`, `setAllowContentAccess(false)`, `setAllowFileAccessFromFileURLs(false)`, `setAllowUniversalAccessFromFileURLs(false)`.

**Testable:** Static analysis verifies WebView settings. Attempt `file://` and `content://` access from WebView. Verify access is denied.

#### PLATFORM-ANDROID-2.2: JavaScript Interface Safety

If `addJavascriptInterface` is used, the app MUST target API 17+ (where `@JavascriptInterface` annotation is required). JavaScript interfaces MUST expose only the minimum necessary methods. The interface MUST validate all input from JavaScript.

**Testable:** Verify `@JavascriptInterface` annotation on all exposed methods. Verify minimum API level >= 17. Review exposed methods for input validation.

#### PLATFORM-ANDROID-2.3: Safe Browsing Enabled

WebViews SHOULD enable Google Safe Browsing via `setSafeBrowsingEnabled(true)` or manifest meta-data to protect against known malicious URLs.

**Testable:** Verify Safe Browsing is enabled. Navigate to a known test phishing URL. Verify warning is displayed.

#### PLATFORM-ANDROID-2.4: URL Allowlisting

WebViews MUST validate loaded URLs against an allowlist. Navigation to unexpected domains MUST be blocked. Consider using `WebViewAssetLoader` for local content.

**Testable:** Attempt to navigate WebView to non-allowlisted URL. Verify navigation is blocked or redirected.

#### PLATFORM-ANDROID-2.5: WebView Data Cleanup on Session End

WebView cache, cookies, form data, and local storage MUST be cleared on logout or session termination.

**Testable:** After logout, inspect WebView storage directories. Verify no residual session data.

### MASVS-PLATFORM-3: Secure Platform Feature Usage

#### PLATFORM-ANDROID-3.1: Verified Deep Links (App Links)

For security-sensitive entry points, the app SHOULD prefer Android App Links (verified deep links via `assetlinks.json`) over custom URL schemes. If custom URL schemes are retained for compatibility or interoperability, they MUST NOT auto-execute sensitive actions and MUST apply the same input validation and user-confirmation requirements as any other untrusted deep link.

**Testable:** Verify `autoVerify="true"` on intent filters. Verify `.well-known/assetlinks.json` is hosted and valid. Verify custom URL scheme is not used for sensitive actions.

#### PLATFORM-ANDROID-3.2: Deep Link Input Validation

All parameters received via deep links MUST be treated as untrusted input and validated/sanitized. Deep links MUST NOT auto-execute sensitive actions (e.g., money transfers, account changes) without additional user confirmation.

**Testable:** Send crafted deep links with malicious parameters. Verify input validation. Verify sensitive actions require user confirmation.

#### PLATFORM-ANDROID-3.3: Minimum Permissions at Point of Use

The app MUST request only the minimum necessary runtime permissions, and MUST request them at the point of use with contextual explanation, not at app launch. The app MUST use `shouldShowRequestPermissionRationale()` to provide context.

**Testable:** Launch app fresh. Verify no permission requests at startup. Trigger feature requiring permission. Verify contextual rationale is shown before request.

#### PLATFORM-ANDROID-3.4: Selected Photos Access

On Android 14+, the app SHOULD use selected photos access (partial media access) instead of requesting full `READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO` permissions when the app only needs access to specific user-selected media.

**Testable:** Verify photo picker or selected photos access is used instead of broad media permissions where applicable.

#### PLATFORM-ANDROID-3.5: QR Code and NFC Input Validation

Decoded QR code and NFC data MUST be treated as untrusted input. The app MUST display decoded URLs to the user before navigating, validate against an allowlist where appropriate, and MUST NOT auto-execute sensitive actions. For payment, authentication, or other high-trust NFC use cases, payload authenticity SHOULD be cryptographically verifiable.

**Testable:** Scan QR code with malicious URL. Verify URL is displayed for user confirmation. Verify non-allowlisted URLs are blocked or warned.

#### PLATFORM-ANDROID-3.6: Tapjacking Protection

Activities displaying sensitive data or accepting sensitive input MUST set `filterTouchesWhenObscured="true"` or check for `FLAG_WINDOW_IS_OBSCURED` to prevent tapjacking (overlay) attacks. Android 16 provides enhanced tapjacking protection.

**Testable:** Attempt tap-through with overlay app on sensitive screen. Verify touches are filtered when obscured.

#### PLATFORM-ANDROID-3.7: Manifest Hardening

The `AndroidManifest.xml` MUST be configured with security-hardened defaults: `android:debuggable="false"` in release builds, `android:allowBackup="false"` or scoped `dataExtractionRules`, Minimal declared permissions, Minimal exported components.

**Testable:** Inspect manifest of release APK. Verify all hardening attributes are correctly set.
