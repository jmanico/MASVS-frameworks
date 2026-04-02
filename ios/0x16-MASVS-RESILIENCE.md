# MASVS-RESILIENCE: Resilience Against Reverse Engineering and Tampering

## Overview

iOS binaries can be reverse-engineered with class-dump, Hopper, Ghidra, and Frida. Jailbreaks (checkra1n, palera1n, Dopamine) remove code signing enforcement and sandbox restrictions. For apps handling financial transactions, DRM, or sensitive business logic, defense-in-depth measures raise the cost of reverse engineering and tampering. This category covers platform integrity validation, anti-tampering, anti-static analysis, and anti-dynamic analysis techniques on iOS.

**Important:** MASVS-RESILIENCE controls are defense-in-depth. They raise the bar but do not provide absolute protection. Critical security logic must always be enforced server-side. These controls complement, not replace, the other MASVS categories.

## iOS Resilience Landscape

| Layer | Threats | Defenses |
|---|---|---|
| Platform integrity | Jailbreak (checkra1n, palera1n, Dopamine), modified runtime | App Attest, multi-signal jailbreak detection |
| App integrity | IPA repackaging, binary patching, dylib injection | Code signing validation, App Attest assertion, dylib detection |
| Static analysis | class-dump, Hopper, Ghidra, strings extraction | Commercial obfuscation (iXGuard, Arxan), symbol stripping |
| Dynamic analysis | Frida, Cycript, lldb, method swizzling | Anti-Frida, anti-debug, timing checks |

## Commercial Tools

| Tool | Type | Capabilities |
|---|---|---|
| Guardsquare iXGuard | Commercial | String encryption, control flow obfuscation, RASP |
| Arxan (Digital.ai) | Commercial | Code hardening, integrity checks, anti-debugging |
| Promon SHIELD | Commercial | Runtime protection, jailbreak detection |
| Zimperium | Commercial | Mobile threat defense, RASP |

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

Running on a jailbroken device removes iOS security guarantees (code signing, sandbox, Keychain protection). Make sure the app detects platform compromise and responds appropriately using App Attest and layered detection.

### iOS Sub-Requirements

#### MASVS-RESILIENCE-1.1 - Use App Attest for Device Attestation

The app uses `DCAppAttestService` (iOS 14+) to generate a hardware-backed key and obtain an attestation from Apple's servers. The server validates the attestation object against Apple's CA, confirming the app is genuine and running on real Apple hardware. All verification happens server-side.

**iOS References:**
- `DCAppAttestService.shared.attestKey(_:clientDataHash:completionHandler:)`
- `DCAppAttestService.shared.generateAssertion(_:clientDataHash:completionHandler:)`

#### MASVS-RESILIENCE-1.2 - Implement Multi-Signal Jailbreak Detection

The app checks multiple signals to detect jailbroken devices. No single check is sufficient.

| Signal | Detection Method |
|---|---|
| URL scheme handlers | `canOpenURL` for Cydia (`cydia://`), Sileo (`sileo://`) |
| Writable system paths | Check for `/private/var/stash`, `/private/var/lib/apt`, `/Applications/Cydia.app` |
| fork() behavior | `fork()` succeeds on jailbroken devices; fails inside the iOS sandbox |
| APT presence | Check for `/etc/apt` existence |
| Dylib injection | Unexpected images via `_dyld_image_count()` and `_dyld_get_image_name()` |
| Environment variables | Check for `DYLD_INSERT_LIBRARIES` and other injection indicators |

**Rationale:** Each individual check can be bypassed. Multiple signals from different categories force attackers to patch every detection path.

#### MASVS-RESILIENCE-1.3 - Detect Jailbreak in Multiple Code Paths

Jailbreak checks run at multiple points during app execution, not just at launch. Checks use different techniques in different locations.

**Rationale:** A single check at launch is trivially patched with a single NOP instruction. Distributed checks across multiple code paths force attackers to find and bypass all of them.

#### MASVS-RESILIENCE-1.4 - Detect Emulators and Simulators

The app detects execution on Simulator by checking `TARGET_OS_SIMULATOR`, `ProcessInfo` properties, and hardware model strings.

**Rationale:** Simulators allow app analysis without hardware restrictions. Attackers can run decrypted IPAs in Simulator to inspect behavior without jailbreaking a physical device.

#### MASVS-RESILIENCE-1.5 - Respond Appropriately to Integrity Failures

The app does not simply crash on jailbreak detection (trivial to bypass). Response strategy depends on risk level:

| Risk Level | Response |
|---|---|
| High-risk apps | Restrict sensitive functionality server-side based on App Attest results |
| Medium-risk apps | Warn the user and log the event |
| All apps | Never reveal which specific check failed |

**Rationale:** A hard crash is a single point of failure. Server-side enforcement based on attestation results is far more resilient than client-side kill switches.

---

## MASVS-RESILIENCE-2

### Control

The app implements anti-tampering mechanisms.

### Description

Without protection, IPA files can be decrypted (from App Store or using frida-ios-dump), modified, and re-signed with a different certificate. Make sure the app detects modifications to its binary and runtime environment.

### iOS Sub-Requirements

#### MASVS-RESILIENCE-2.1 - Validate Code Signing at Runtime

The app verifies its `embedded.mobileprovision` and code signature at runtime. Repackaged apps will have a different signing identity.

**Rationale:** IPA repackaging with a different certificate is a common attack vector. Runtime verification of the expected signing identity detects modified binaries.

#### MASVS-RESILIENCE-2.2 - Use App Attest Assertions Per Request

For sensitive API calls, the app generates an App Attest assertion including a server nonce. The server validates the assertion to confirm the same attested app instance is making the request.

**iOS References:**
- `DCAppAttestService.shared.generateAssertion(_:clientDataHash:completionHandler:)`

**Rationale:** A one-time attestation at launch can be replayed. Per-request assertions bind each API call to the attested app instance and prevent request forgery from modified clients.

#### MASVS-RESILIENCE-2.3 - Detect Dylib Injection

The app inspects loaded dynamic libraries using `_dyld_image_count()` and `_dyld_get_image_name()` to detect injected libraries (Frida gadget, Cycript, custom dylibs).

**Rationale:** Dylib injection is the primary method for runtime manipulation on jailbroken iOS. Frida operates by injecting `FridaGadget.dylib` into the target process.

#### MASVS-RESILIENCE-2.4 - Use DeviceCheck for Server-Side State

The app uses `DeviceCheck` to maintain 2 bits of per-device state on Apple's servers. Useful for fraud detection, trial abuse prevention, and marking compromised devices.

**iOS References:**
- `DCDevice.current.generateToken(completionHandler:)`

**Rationale:** DeviceCheck state persists across app reinstalls and device restores. Attackers cannot reset this state by reinstalling the app.

#### MASVS-RESILIENCE-2.5 - Report Tampering to Server

When tampering is detected, the app reports the event to the server. The server decides enforcement action (block, limit, or monitor). The app does not silently continue operating in a compromised state.

**Rationale:** Client-side enforcement is always bypassable. Server-side decisions based on tampering telemetry allow graduated responses and threat intelligence collection.

---

## MASVS-RESILIENCE-3

### Control

The app implements anti-static analysis mechanisms.

### Description

iOS has no built-in obfuscation equivalent to Android's R8. Reverse engineers use class-dump, Hopper, and Ghidra on decrypted IPAs to reconstruct app logic. Commercial tools provide obfuscation for high-value apps.

### iOS Sub-Requirements

#### MASVS-RESILIENCE-3.1 - Strip Debug Symbols from Release Builds

Release builds strip debug symbols using the following Xcode build settings:

| Build Setting | Value |
|---|---|
| `STRIP_INSTALLED_PRODUCT` | `YES` |
| `DEPLOYMENT_POSTPROCESSING` | `YES` |
| `STRIP_STYLE` | `all` |

Debug information is not included in the distributed binary.

**Rationale:** Debug symbols map memory addresses to function names, variable names, and source file locations. Stripping them forces reverse engineers to work with raw disassembly.

#### MASVS-RESILIENCE-3.2 - Use Commercial Obfuscation for High-Value Apps

Apps handling financial transactions, DRM, or sensitive business logic use commercial obfuscation (iXGuard, Arxan) for string encryption, control flow obfuscation, and class/method renaming.

**Rationale:** Without obfuscation, class-dump produces readable Objective-C headers and Swift metadata is easily parsed. Commercial tools significantly increase the time required for static analysis.

#### MASVS-RESILIENCE-3.3 - Encrypt Sensitive String Literals

Sensitive strings (API endpoints, algorithm names, key aliases) are not present as plaintext in the binary. Use build-time string encryption or commercial tools.

**Rationale:** The `strings` command trivially extracts all string literals from a Mach-O binary. Plaintext strings reveal API URLs, cryptographic algorithm names, and internal logic.

#### MASVS-RESILIENCE-3.4 - Remove Verbose Logging from Release Builds

Release builds exclude debug logging via build configuration (`#if DEBUG`) or compiler optimization. No verbose log statements are present in the production binary.

**Rationale:** Log strings reveal internal logic, state transitions, error handling paths, and variable names. They provide a roadmap for reverse engineers.

---

## MASVS-RESILIENCE-4

### Control

The app implements anti-dynamic analysis techniques.

### Description

Dynamic analysis via Frida, Cycript, lldb, and method swizzling allows runtime manipulation of any app function. Make sure the app detects and resists instrumentation.

### iOS Sub-Requirements

#### MASVS-RESILIENCE-4.1 - Detect Debugger Attachment

The app detects debuggers via `sysctl` (checking the `P_TRACED` flag) and `ptrace` (`PT_DENY_ATTACH`).

```c
// sysctl-based detection
struct kinfo_proc info;
size_t info_size = sizeof(info);
int name[4] = {CTL_KERN, KERN_PROC, KERN_PROC_PID, getpid()};
sysctl(name, 4, &info, &info_size, NULL, 0);
if (info.kp_proc.p_flag & P_TRACED) { /* debugger detected */ }
```

**Rationale:** Debuggers allow memory inspection, breakpoints, and variable modification at runtime. `PT_DENY_ATTACH` prevents debugger attachment; `sysctl` detects an already-attached debugger.

#### MASVS-RESILIENCE-4.2 - Detect Frida and Cycript

The app detects Frida and Cycript through multiple signals:

| Signal | Detection Method |
|---|---|
| Frida server process | Check for `frida-server` in running processes |
| Frida gadget dylib | Scan loaded dylibs for `FridaGadget` |
| Named pipes | Check for Frida-specific named pipes in `/tmp` |
| Frida ports | Check for listeners on default Frida port (27042) |
| Cycript dylib | Scan loaded dylibs for `cycript` |

**Rationale:** Frida is the primary tool for iOS runtime manipulation. Detection of multiple Frida artifacts makes bypass more difficult.

#### MASVS-RESILIENCE-4.3 - Implement Timing-Based Detection

The app measures execution time of critical code sections and flags anomalies indicating single-stepping or breakpoint activity.

```swift
let start = mach_absolute_time()
// critical code section
let elapsed = mach_absolute_time() - start
if elapsed > expectedThreshold { /* possible debugging */ }
```

**Rationale:** Debuggers introduce measurable latency when single-stepping or when breakpoints trigger. Timing anomalies of 10x or more indicate instrumentation.

#### MASVS-RESILIENCE-4.4 - Detect Method Swizzling

The app verifies that critical method implementations have not been swizzled by comparing IMP pointers against expected values.

**Rationale:** Method swizzling replaces method implementations at runtime by modifying the Objective-C dispatch table. Attackers use swizzling to intercept authentication, encryption, and validation methods.

#### MASVS-RESILIENCE-4.5 - Integrate RASP for High-Security Apps

High-security apps integrate Runtime Application Self-Protection (RASP) solutions for continuous runtime threat detection covering jailbreak, debugging, hooking, and emulator detection in a single SDK.

**Rationale:** Individual detection checks require ongoing maintenance as bypass techniques evolve. RASP solutions from vendors (Promon SHIELD, Zimperium, Guardsquare) provide continuously updated detection across all threat categories.

---

## Training-Aligned Requirements

| ID | Requirement | Level | Testable Via |
|---|---|---|---|
| RESILIENCE-IOS-1.1 | The app MUST use `DCAppAttestService` for device/app attestation. Server MUST validate attestation objects. | MUST | Verify App Attest integration and server-side validation |
| RESILIENCE-IOS-1.2 | The app SHOULD detect jailbroken devices via multiple signals. Server-side enforcement MUST be primary. | SHOULD / MUST | Run app on jailbroken device and verify detection |
| RESILIENCE-IOS-1.3 | High-security apps SHOULD detect Frida attachment. | SHOULD | Attach Frida to running app and verify detection fires |
| RESILIENCE-IOS-2.1 | The app SHOULD verify its signing identity at runtime. | SHOULD | Re-sign IPA with a different certificate and verify detection |
| RESILIENCE-IOS-2.2 | The app SHOULD detect injected dylibs. | SHOULD | Inject a test dylib and verify detection |
| RESILIENCE-IOS-3.1 | Release builds MUST strip debug symbols. | MUST | Run `nm` on the release binary and confirm no debug symbols |
| RESILIENCE-IOS-3.2 | Sensitive strings SHOULD be encrypted in the binary. | SHOULD | Run `strings` on the binary and search for known sensitive values |
| RESILIENCE-IOS-4.1 | The app SHOULD detect debugger attachment. | SHOULD | Attach `lldb` and verify detection response |
| RESILIENCE-IOS-4.2 | High-security apps SHOULD integrate a RASP SDK. | SHOULD | Verify RASP SDK is present and active at runtime |
