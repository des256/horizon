# ohmu-mobile

Platform bridge for Android and iOS. May be split into `ohmu-android` and `ohmu-ios` if the implementations diverge significantly, but starts as one crate with conditional compilation.

## What lives here

### Android

- **Activity/Surface integration** — `NativeActivity` or `GameActivity` (via `android-activity` crate) providing an `ANativeWindow` for wgpu surface creation.
- **Input translation** — `AInputEvent` (touch, key, stylus) to `ohmu-core` events. Handles multi-touch pointer IDs, pressure/tilt for stylus.
- **IME** — `InputConnection` implementation for software keyboard, composition, autocorrect. Communicates cursor rect for popup positioning.
- **Clipboard** — `ClipboardManager` via JNI.
- **Text shaping** — links system HarfBuzz (Android ships it) or bundles `rustybuzz`.
- **Accessibility** — `AccessibilityNodeInfo` via `AccessibilityDelegate` for TalkBack support.
- **Lifecycle** — handles `onPause`/`onResume`/`onDestroy`, surface created/changed/destroyed, configuration changes (rotation, locale).
- **TargetProfile detection** — `DisplayMetrics` for density/size, `Configuration` for orientation, `InputDevice` capabilities, accessibility settings from `Settings.Secure`.

### iOS

- **UIKit/Metal integration** — `CAMetalLayer` on a `UIView` for wgpu surface (Metal backend). App delegate and view controller in Rust via `objc2` bindings or thin Swift/ObjC bridge.
- **Input translation** — `UITouch`/`UIPress` events to `ohmu-core` events. Handles multi-touch, Apple Pencil (pressure, tilt, azimuth).
- **IME** — `UITextInput` protocol implementation for software keyboard, dictation, autocorrect, smart punctuation.
- **Clipboard** — `UIPasteboard`.
- **Text shaping** — Core Text for shaping (platform-native) or bundled `rustybuzz`.
- **Accessibility** — `UIAccessibility` protocol for VoiceOver support.
- **Lifecycle** — `UIApplicationDelegate` lifecycle, `UIScene` for multi-window iPad.
- **TargetProfile detection** — `UIScreen.main.bounds`/`scale`, `UIDevice` for form factor, `traitCollection` for size classes, safe area insets, Dynamic Type scale, accessibility settings.

## Dependencies

- `ohmu-core`, `ohmu-gpu`, `ohmu-dsl`, `ohmu-platform`.
- `wgpu` — Vulkan on Android, Metal on iOS.
- Android: `android-activity`, `ndk`, `jni` crates.
- iOS: `objc2`, `block2` crates (or thin ObjC bridge code).

## Build

```bash
# Android (via cargo-ndk)
cargo ndk -t arm64-v8a -t armeabi-v7a build --release -p ohmu-mobile

# iOS
cargo build --target aarch64-apple-ios --release -p ohmu-mobile          # device
cargo build --target aarch64-apple-ios-sim --release -p ohmu-mobile      # simulator
```

## Testing tiers

### Tier 1 — Pure logic (inherited)

All `ohmu-core` and `ohmu-dsl` tests cross-compile and run on-device or in-emulator. Verifies WASM-free correctness on ARM architectures and mobile stdlib behavior.

```bash
# Android: push and run test binary
cargo ndk -t arm64-v8a test -p ohmu-core --no-run
adb push target/aarch64-linux-android/debug/deps/ohmu_core-* /data/local/tmp/
adb shell /data/local/tmp/ohmu_core-*

# iOS simulator
cargo build --target aarch64-apple-ios-sim --tests -p ohmu-core
xcrun simctl spawn booted target/aarch64-apple-ios-sim/debug/deps/ohmu_core-*
```

### Tier 2 — GPU on-device

Run compute shader verification and visual regression on mobile GPUs (Adreno, Mali, Apple GPU). Same tests as `ohmu-gpu` Tier 2 but on actual mobile hardware to catch driver-specific issues.

Mobile GPUs have tighter limits (max buffer sizes, workgroup sizes, precision) and different performance characteristics. Floating-point precision differences are the most common source of CPU/GPU layout mismatches on mobile.

### Tier 3 — Full device integration (primary)

**Android:**
- Build APK with test harness.
- Deploy via `adb install`.
- Run instrumented tests or UI automation via `uiautomator2`.
- Verify: touch input, software keyboard + IME (test CJK composition), screen rotation, split-screen multitasking, TalkBack accessibility.
- Screenshot capture via `adb shell screencap` for visual comparison.

**iOS:**
- Build test app, deploy to simulator or device.
- UI automation via XCTest or `simctl`.
- Verify: touch input, iOS keyboard + IME, rotation, iPad multitasking (Slide Over, Split View), VoiceOver, Dynamic Type scaling.
- Screenshot capture via `simctl io screenshot`.

**Device matrix (target coverage):**

| Device class | Android | iOS |
|-------------|---------|-----|
| Phone | Pixel (Adreno), Galaxy (Mali) | iPhone (Apple GPU) |
| Tablet | Galaxy Tab (Mali) | iPad (Apple GPU) |
| Foldable | Galaxy Fold (fold state changes) | — |
| Low-end | Budget device (limited GPU, small RAM) | iPhone SE |
