# Amethyst iOS 25 – Pull Request Summary

## Changes Overview

This PR fixes two major compatibility issues (Sodium 0.6+ render blocking and Simple Voice Chat native library loading) and adds a native microphone source selection UI.

## Files Modified

| File | Purpose |
|------|---------|
| `JavaApp/src/patchjna_agent/com/sun/jna/Platform.java` | Voice Chat + Sodium compatibility (JNA `isMac()` override using string-based class name matching) |
| `JavaApp/src/patchjna_agent/net/kdt/patchjna/PatchJNAAgent.java` | Sodium launcher/LWJGL checks bypass via bytecode instrumentation |
| `Natives/SurfaceViewController.m` | Microphone source selection logic (`selectMicrophoneSource`) |
| `Natives/LauncherPreferencesViewController.m` | Microphone source preferences entry |
| `Natives/PLPreferences.m` | Default value for `microphone_source` |
| `Natives/resources/en.lproj/Localizable.strings` | Mic labels (English) |
| `Natives/CMakeLists.txt` | Fix UnzipKit include path |
| `Makefile` | Build fix for actool + SPIRV_CROSS_SHARED |

## 1. Sodium Fix – "This launcher is not supported"

**Problem:** Sodium 0.6+ detects PojavLauncher via `PostLaunchChecks.isUsingPojavLauncher()` and blocks rendering. It also checks the LWJGL version in `PreLaunchChecks.isUsingKnownCompatibleLwjglVersion()`.

**Fix:** Bytecode-level method patching in `PatchJNAAgent.java`:
- `PostLaunchChecks.isUsingPojavLauncher()` → returns `false`
- `PreLaunchChecks.isUsingKnownCompatibleLwjglVersion()` → returns `true`

## 2. Simple Voice Chat Fix – "Unsupported macOS launcher" + "Some modules failed to load"

**Problem:** Simple Voice Chat checks `Platform.isMac()` in multiple places. On iOS, PojavLauncher's JNA reports `MAC` as the OS type, so `isMac()` returns `true`. This causes:
- `MicrophoneManager` constructor throws `UnsupportedOperationException`
- `ClientManager.checkMicrophonePermissions()` shows "Your launcher does not support macOS microphone permissions"
- `NativeValidator` skips native library loading when `VersionCheck` returns false

**Fix (Platform.java):** Replace the old `Class.forName()` approach (which fails for Fabric mods loaded by different classloaders) with a static `Set<String>` of class names. `isMac()` returns `false` only for listed voice-chat classes:

| Caller | `isMac()` returns | Effect |
|--------|-------------------|--------|
| `NativeValidator` | `false` | Skips macOS version check → **native libs load** |
| `VersionCheck` | `false` | `isMacOSNativeCompatible()` returns `false` → permission check suppressed |
| `MicrophoneManager` | `false` | Constructor does **not throw** |
| Everything else | `true` | Normal behavior |

## 3. Microphone Source Selection

Added a native 4-option picker (Preferences → Audio):
- **Auto (Recommended)** – tries Front → Bottom → Back
- **Front microphone** / **Bottom microphone** / **Back microphone**

Uses `AVAudioSession` to select the physical data source on the device.

## Build Notes

- Requires JDK 8 (`BOOTJDK`), cmake, ldid, and Xcode with iOS SDK
- Full build: `make PLATFORM=2 RELEASE=1`
- The `-javaagent:patchjna_agent.jar` is injected by `JavaLauncher.m`

## Bug notice

When you first launch the launcher, please rotate to landscape AFTER the home screen appears. If you rotate to landscape before the home screen fully loads, the game will crash after the Mojang logo.