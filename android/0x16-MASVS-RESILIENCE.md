# MASVS-RESILIENCE: Resilience Against Reverse Engineering and Tampering

## Overview

Android APKs are trivially decompilable - `jadx` produces readable Java/Kotlin from DEX bytecode, `apktool` extracts resources and manifest, and tools like Frida and Xposed allow runtime manipulation of any app function. For apps that handle financial transactions, DRM, anti-cheat, or sensitive business logic, defense-in-depth measures are necessary to increase the cost and complexity of reverse engineering and tampering. This category covers platform integrity validation, anti-tampering, anti-static analysis, and anti-dynamic analysis techniques on Android.

**Important:** MASVS-RESILIENCE controls are defense-in-depth. They raise the bar but do not provide absolute protection. Critical security logic must always be enforced server-side. These controls complement, not replace, the other MASVS categories.

## Android Resilience Landscape

| Layer | Threats | Defenses |
|---|---|---|
| Platform integrity | Root (Magisk/KernelSU), custom ROM, unlocked bootloader | Play Integrity API, multi-signal root detection, emulator detection |
| App integrity | APK repackaging, DEX patching, resource modification | Signing verification, Play Integrity app verdict, checksum verification |
| Static analysis | jadx decompilation, strings extraction, manifest analysis | R8 obfuscation, string encryption, control flow obfuscation, native obfuscation |
| Dynamic analysis | Frida hooking, Xposed modules, debugger attachment | Anti-Frida, anti-debug, anti-hook, timing checks |
| Runtime environment | Screen capture, accessibility abuse, overlay attacks | Play Integrity app access risk, FLAG_SECURE, tapjacking protection |

## Commercial vs. Open-Source Tools

| Tool | Type | Capabilities |
|---|---|---|
| R8 (Google) | Free | Identifier renaming, dead code removal, optimization |
| Guardsquare DexGuard | Commercial | String encryption, class encryption, control flow obfuscation, RASP |
| Promon SHIELD | Commercial | Runtime protection, integrity checks, anti-debugging, anti-hooking |
| Verimatrix | Commercial | Code hardening, white-box cryptography, app shielding |
| O-LLVM / Hikari | Open-source | Native code obfuscation (LLVM-based) |
| RootBeer | Open-source | Root detection library (community-maintained) |

## Controls Summary

| Control | Title |
|---|---|
| MASVS-RESILIENCE-1 | The app validates the integrity of the platform |
| MASVS-RESILIENCE-2 | The app implements anti-tampering mechanisms |
| MASVS-RESILIENCE-3 | The app implements anti-static analysis mechanisms |
| MASVS-RESILIENCE-4 | The app implements anti-dynamic analysis techniques |

## OWASP Mobile Top 10 2024 Mapping

- **M7 - Insufficient Binary Protections:** Directly addressed by all four controls

---

## MASVS-RESILIENCE-1

### Control

The app validates the integrity of the platform.

### Description

Running on a compromised Android platform - rooted devices, unlocked bootloaders, custom ROMs, or devices with Magisk/Xposed - undermines many security controls that depend on platform guarantees (sandboxing, secure storage, biometric integrity). Make sure the app detects platform compromise and responds appropriately, using multiple layered detection techniques and server-side verification via the Google Play Integrity API.

### Android Sub-Requirements

#### MASVS-RESILIENCE-1.1 - Use Google Play Integrity API for Device Attestation

The app requests a Play Integrity token on launch or before sensitive operations and sends it to the server for verification. The server evaluates:

| Verdict | Label | Action |
|---|---|---|
| `deviceRecognitionVerdict` | `MEETS_DEVICE_INTEGRITY` | Allow full functionality |
| `deviceRecognitionVerdict` | `MEETS_BASIC_INTEGRITY` only | Warn user; restrict sensitive features |
| `deviceRecognitionVerdict` | No labels | Block sensitive operations |
| `appRecognitionVerdict` | `PLAY_RECOGNIZED` | App is genuine |
| `appRecognitionVerdict` | Not recognized | App may be modified; restrict |
| `appLicensingVerdict` | `LICENSED` | App acquired from Play Store |

**Critical:** All verdict evaluation must happen server-side.

**Android References:**
- `com.google.android.play:integrity`
- `IntegrityManager.requestIntegrityToken(IntegrityTokenRequest)`

#### MASVS-RESILIENCE-1.2 - Implement Multi-Signal Root Detection

In addition to Play Integrity, the app performs local root detection checks as defense-in-depth:

| Signal | Detection Method |
|---|---|
| `su` binary | Check `/system/bin/su`, `/system/xbin/su`, `/sbin/su`, `/system/app/Superuser.apk` |
| Root management apps | Check for Magisk Manager, SuperSU, KingRoot packages via `PackageManager` |
| Build properties | Check `ro.build.tags` for "test-keys", `ro.debuggable` for "1" |
| SELinux status | Verify SELinux is in "enforcing" mode |
| System partition | Check `/system` is mounted read-only |
| Mount points | Parse `/proc/mounts` for Magisk overlay mounts |
| Xposed/LSPosed | Check for Xposed Installer package; check `/proc/self/maps` for Xposed libraries |

**Important:** Local root detection is a speed bump, not a security boundary.

#### MASVS-RESILIENCE-1.3 - Perform Root Checks in Native Code

Critical root detection logic is implemented in native code (C/C++ via JNI) in addition to Java/Kotlin, cross-referencing results from both layers.

**Rationale:** Java/Kotlin method calls are trivially hookable with Frida or Xposed. Native code requires more sophisticated instrumentation to bypass.

#### MASVS-RESILIENCE-1.4 - Detect Emulators

The app detects execution on emulators by checking:

- `Build.FINGERPRINT` contains "generic", "sdk", "google_sdk"
- `Build.MODEL` contains "Emulator", "Android SDK", "sdk_gphone"
- Emulator-specific files exist (`/dev/qemu_pipe`, `/dev/goldfish_pipe`)
- `TelephonyManager` returns known-fake IMEI/IMSI values
- Hardware sensors return unrealistic or absent readings

**Rationale:** Emulators allow easy app instrumentation, debugging, and traffic interception.

#### MASVS-RESILIENCE-1.5 - Respond Appropriately to Platform Integrity Failures

When platform integrity checks fail, the app does not simply display an error and exit. Instead:

- **High-risk apps:** restrict sensitive functionality server-side based on Play Integrity verdicts
- **Medium-risk apps:** warn the user and log the event
- **All apps:** never reveal which specific check failed

**Rationale:** Binary "pass/fail" responses are easy to bypass by hooking the single decision point.

---

## MASVS-RESILIENCE-2

### Control

The app implements anti-tampering mechanisms.

### Description

Without protection, it is relatively straightforward to modify an Android APK - re-signing with a different key after code or resource modification, patching DEX bytecode, or uploading a backdoored version to third-party stores. Make sure the app detects modifications to its own code, resources, and signing certificate to maintain integrity.

### Android Sub-Requirements

#### MASVS-RESILIENCE-2.1 - Verify App Signing Certificate at Runtime

The app verifies its own signing certificate at runtime against a hardcoded expected certificate hash.

**Rationale:** Repackaged APKs must be signed with a different key.

**Android References:**
- `PackageManager.getPackageInfo()` with `GET_SIGNING_CERTIFICATES`

#### MASVS-RESILIENCE-2.2 - Verify APK Integrity via Play Integrity

Use the Play Integrity API's `appRecognitionVerdict` to verify server-side that the APK matches the version published on Google Play (`PLAY_RECOGNIZED`).

**Rationale:** Client-side integrity checks can be patched out.

#### MASVS-RESILIENCE-2.3 - Detect DEX and Native Library Modifications

The app computes checksums of its DEX files and native libraries at runtime and compares them against expected values.

**Rationale:** Attackers modify DEX bytecode to bypass security checks.

#### MASVS-RESILIENCE-2.4 - Use v2+ APK Signing Scheme

The app is signed with APK Signature Scheme v2 or v3 (not v1 JAR signing alone).

**Android References:**
- Use `apksigner`
- v3 signing (API 28+) enables key rotation
- v3.1 (API 33+)

#### MASVS-RESILIENCE-2.5 - Detect Installation Source

The app verifies its installation source using `PackageManager.getInstallSourceInfo()` (API 30+). Apps distributed via Google Play should verify the installer is `com.android.vending`.

**Rationale:** Apps sideloaded from third-party sources may have been modified.

#### MASVS-RESILIENCE-2.6 - Implement Redundant Integrity Checks

Integrity checks are implemented in multiple locations throughout the app's execution flow and use different techniques.

**Rationale:** A single integrity check at startup is trivially bypassed.

---

## MASVS-RESILIENCE-3

### Control

The app implements anti-static analysis mechanisms.

### Description

Understanding the internals of an Android app through static analysis is typically the first step toward tampering. Make sure the app makes static analysis significantly more difficult.

### Android Sub-Requirements

#### MASVS-RESILIENCE-3.1 - Enable R8 Code Shrinking and Obfuscation

R8 is enabled for all release builds with `minifyEnabled true` and `shrinkResources true`.

**Rationale:** R8 renames classes, methods, and fields to short, meaningless names.

#### MASVS-RESILIENCE-3.2 - Encrypt String Literals Containing Sensitive Values

Sensitive strings are not present as plaintext in the DEX file.

**Rationale:** R8 does NOT encrypt strings.

#### MASVS-RESILIENCE-3.3 - Apply Control Flow Obfuscation (High-Value Apps)

For high-value apps, use commercial obfuscation tools that implement control flow obfuscation.

**Rationale:** R8 only performs identifier renaming.

**Commercial Tools:**
- Guardsquare DexGuard
- Promon SHIELD
- Verimatrix

#### MASVS-RESILIENCE-3.4 - Obfuscate Native Code

If the app contains native libraries, native code is obfuscated using LLVM-based obfuscators with:

- Control flow flattening
- Instruction substitution
- String encryption
- Symbol stripping

**Rationale:** Native code is decompilable with Ghidra or IDA Pro.

#### MASVS-RESILIENCE-3.5 - Remove Debug Information from Release Builds

Release builds are configured with:

- `android:debuggable="false"`
- Debug symbols stripped
- Log statements removed via R8 rules
- No `BuildConfig.DEBUG` code paths active

**Rationale:** Debug information makes reverse engineering significantly easier.

---

## MASVS-RESILIENCE-4

### Control

The app implements anti-dynamic analysis techniques.

### Description

Dynamic analysis - attaching debuggers, hooking functions with Frida, instrumenting with Xposed/LSPosed, and runtime manipulation - allows attackers to modify app behavior. Make sure the app detects and resists dynamic instrumentation.

### Android Sub-Requirements

#### MASVS-RESILIENCE-4.1 - Detect Debugging

The app detects when a debugger is attached:

- Check `android.os.Debug.isDebuggerConnected()`
- Check `TracerPid` in `/proc/self/status`
- Use `ptrace(PTRACE_TRACEME, 0, 0, 0)` in native code
- Monitor timing anomalies

**Rationale:** Debuggers allow inspection and modification of memory and variables.

#### MASVS-RESILIENCE-4.2 - Detect Frida and Dynamic Instrumentation Frameworks

The app detects Frida, Xposed, and other instrumentation frameworks:

| Framework | Detection Technique |
|---|---|
| Frida (server) | Check for Frida default port 27042; scan `/proc/self/maps` for `frida-agent`, `frida-gadget` |
| Frida (gadget) | Check for Frida-specific named threads (`gmain`, `gdbus`, `gum-js-loop`) in `/proc/self/task/*/comm` |
| Frida (hooks) | Verify integrity of function prologues in libc |
| Xposed/LSPosed | Check for Xposed Installer/LSPosed Manager packages; scan `/proc/self/maps` for Xposed bridge libraries |
| Magisk (Zygisk) | Check for Zygisk modules in `/data/adb/modules/`; detect injected libraries in `/proc/self/maps` |

**Rationale:** Frida is the most widely used dynamic instrumentation tool.

#### MASVS-RESILIENCE-4.3 - Implement Anti-Hook Techniques

The app verifies the integrity of critical security functions:

- Compare function prologue bytes
- Use `dlsym()` to resolve function addresses
- Implement critical checks using raw syscalls
- Use JNI `RegisterNatives()`

**Rationale:** Frida works by replacing function prologues with trampolines.

#### MASVS-RESILIENCE-4.4 - Detect Runtime Environment Manipulation

The app detects signs of runtime manipulation:

- Check `Settings.Global.ADB_ENABLED`
- Verify `ApplicationInfo.FLAG_DEBUGGABLE`
- Detect suspicious environment variables
- Monitor for unexpected loaded libraries in `/proc/self/maps`

#### MASVS-RESILIENCE-4.5 - Implement Timing-Based Detection

The app uses timing checks to detect single-stepping and breakpoint-based debugging.

**Rationale:** Debuggers and instrumentation tools introduce latency.

#### MASVS-RESILIENCE-4.6 - Use App Access Risk Verdict (Android 15+)

For apps handling sensitive data, use Play Integrity API's `appAccessRiskVerdict` to detect:

- `UNKNOWN_CAPTURING`
- `UNKNOWN_CONTROLLING`
- `UNKNOWN_OVERLAYS`

**Rationale:** Screen capture and accessibility-based control apps can exfiltrate data.

**Android References:**
- `environmentDetails.appAccessRiskVerdict`
- Available for apps distributed via Google Play with `MEETS_DEVICE_INTEGRITY`

---

## Training-Aligned Requirements

### MASVS-RESILIENCE-1: Reverse Engineering Resistance

**RESILIENCE-ANDROID-1.1: Code Obfuscation with R8**
Release builds MUST enable R8 with code shrinking, obfuscation, and optimization.
**Testable:** Decompile release APK with jadx. Verify class/method names are obfuscated.

**RESILIENCE-ANDROID-1.2: Native Code Protection**
If the app uses native code, native libraries SHOULD be stripped of debug symbols and SHOULD use additional obfuscation.
**Testable:** Run `nm` or `readelf` on `.so` files.

**RESILIENCE-ANDROID-1.3: String Encryption**
Sensitive strings SHOULD be encrypted or obfuscated in the APK.
**Testable:** Search decompiled APK for known sensitive strings.

**RESILIENCE-ANDROID-1.4: Server-Side Sensitive Logic**
Sensitive business logic MUST be implemented server-side.
**Testable:** Review app architecture. Verify sensitive decisions are made server-side.

### MASVS-RESILIENCE-2: Tamper Detection and Response

**RESILIENCE-ANDROID-2.1: Play Integrity API**
The app MUST use the Play Integrity API to verify: App integrity, Device integrity, Account licensing. Attestation results MUST be verified server-side.
**Testable:** Verify Play Integrity API integration. Verify attestation token is sent to server.

**RESILIENCE-ANDROID-2.2: Root/Magisk Detection**
The app SHOULD detect rooted devices via multiple signals: Presence of `su` binary, Magisk/Zygisk artifacts, Modified system partitions, Play Integrity `MEETS_STRONG_INTEGRITY` tier. Root detection MUST be verified server-side.
**Testable:** Run app on rooted device with Magisk.

**RESILIENCE-ANDROID-2.3: Frida Detection**
For high-security apps, the app SHOULD detect common dynamic instrumentation tools via: Process name inspection, Named pipe detection, Code section hash verification, Frida gadget detection.
**Testable:** Attach Frida to the app process.

**RESILIENCE-ANDROID-2.4: Tamper Response**
When tampering is detected, the app MUST: Report the violation to the server, The server MUST decide enforcement action, The app SHOULD NOT silently continue in compromised state.
**Testable:** Trigger integrity violation. Verify server receives report.

### MASVS-RESILIENCE-3: File Integrity

**RESILIENCE-ANDROID-3.1: APK Integrity Verification**
The app SHOULD verify its own APK signature at runtime.
**Testable:** Repackage and resign APK with different key.

**RESILIENCE-ANDROID-3.2: Resource Integrity**
Critical app resources SHOULD have integrity checks.
**Testable:** Modify a critical resource file.

### MASVS-RESILIENCE-4: Client-Side Security Controls

**RESILIENCE-ANDROID-4.1: RASP Integration**
High-security apps SHOULD integrate RASP solutions.
**Testable:** Verify RASP SDK integration.

**RESILIENCE-ANDROID-4.2: Debugger Detection**
The app SHOULD detect debugger attachment.
**Testable:** Attach a debugger.

**RESILIENCE-ANDROID-4.3: Emulator Detection**
The app SHOULD detect execution on emulators.
**Testable:** Run app on emulator.
