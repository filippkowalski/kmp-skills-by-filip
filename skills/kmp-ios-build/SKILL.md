---
name: kmp-ios-build
description: Build, link and ship the iOS host of a Kotlin Multiplatform + Compose Multiplatform app. Use when the iOS test binary fails to link, when Swift and Kotlin pass data or callbacks, when a CI archive runs out of heap in the Kotlin framework phase, when App Store Connect refuses an upload for host configuration, when setting up XcodeGen, entitlements, iPad scenes or quick actions, or when a sign-in survives an iOS reinstall. Triggers - iosApp, project.yml, xcodegen, embedAndSignAppleFrameworkForXcode, ComposeUIViewController, MainViewController, Swift bridge, completionHandler, suspend from Swift, iosSimulatorArm64Test, linker error, __swift_FORCE_LOAD, framework not found, Java heap space, Codemagic, ITMS-90476, ITMS-90683, UILaunchScreen, aps-environment, SceneDelegate, quick action, Keychain, reinstall, anonymous sign-in.
---

# iOS host for KMP + Compose Multiplatform

Most expensive traps first.

## 1. If a Swift bridge forwards SDK data to Kotlin: a callback typed `Any` may carry a Swift value

- **Symptom:** iOS only. Messages from a native SDK never reach Kotlin. No error, no log.
- **Cause:** the SDK decodes its payload into its own `Codable` enum or struct before it calls
  `callback(data: Any)`. `JSONSerialization.isValidJSONObject(data)` is false for any Swift value
  type, so a guard built on it drops every message.
- **Fix:** send JSON strings across the Swift/Kotlin boundary. Re-encode `Codable` values with
  `JSONEncoder`. Log every drop on both sides of the bridge.
- **Seen on:** a third-party voice SDK for iOS (pipecat-client-ios 1.3.0), Kotlin 2.1.21.

```swift
func onMessage(data: Any) {
    guard let value = data as? SDKValue, let d = try? JSONEncoder().encode(value),
          let json = String(data: d, encoding: .utf8)
    else { NSLog("bridge: dropped %@", String(describing: type(of: data))); return }
    kotlinListener.onMessage(json: json)
}
```

## 2. If a dependency ships Swift or SPM frameworks: the iOS test executable links without Xcode's additions

- **Symptom:** `iosSimulatorArm64Test` fails to link while the app builds: `framework not found
  <Name>`, or undefined `__swift_FORCE_LOAD_$_swiftCompatibility56`.
- **Cause:** Gradle links the test executable itself. Xcode links the app and adds SPM frameworks
  and the Swift library search paths. A cinterop that emits `-framework X` works only where SPM
  provides X. A klib that embeds a compiled Swift SDK can carry linker options with an absolute
  Xcode path from the vendor's build machine, which exists nowhere else.
- **Fix:** give the test link, and only the test link, what Xcode gives the app. For embedded
  Swift, add this machine's Swift simulator libraries. Resolve the path lazily, so an Android-only
  build on Linux never runs `xcode-select`. For a `-framework` flag no test uses, an empty dylib
  of that name plus `-rpath` links. If a klib embeds a native SDK, do not also add it via SPM.
- **Seen on:** purchases-kmp 2.10.2 (`-framework PurchasesHybridCommon`) and 3.5.0 (embedded
  `libRevenueCat.a`), Kotlin 2.1.21 and 2.4.10.

```kotlin
val swiftSimLibs = providers.exec { commandLine("xcode-select", "-p") }.standardOutput.asText
    .map { listOf("-linker-option",
        "-L${it.trim()}/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/iphonesimulator") }
kotlin { iosSimulatorArm64 {
    binaries.withType<org.jetbrains.kotlin.gradle.plugin.mpp.TestExecutable>().configureEach {
        linkTaskProvider.configure { toolOptions.freeCompilerArgs.addAll(swiftSimLibs) }
    }
} }
```

See also: kotlin-tooling-cocoapods-spm-migration in Kotlin/kotlin-agent-skills (its troubleshooting reference covers `framework not found` from klibs with baked `-framework` flags).

## 3. If iOS builds on CI: the Kotlin/Native compiler heap is the Gradle daemon heap

- **Symptom:** the archive fails in the Xcode build phase that runs Gradle, with
  `Compilation failed: Java heap space`. Raising `kotlin.native.jvmArgs` changes nothing.
- **Cause:** the Kotlin/Native compiler runs inside the Gradle daemon, so `org.gradle.jvmargs`
  sets its heap. On a 12 GB build machine, 4 GB and 2 GB were both too small.
- **Fix:** write CI-only values to `~/.gradle/gradle.properties` on the builder. They override the
  project file, so local values stay as they are. On 12 GB, 7 GB leaves room for Xcode. Pin the
  CI Xcode to your local one. Cache `~/.konan`, `~/.gradle/caches`, `~/.gradle/wrapper`.
  Kotlin/Native builds Apple targets only on macOS, so iOS CI needs a macOS runner. The `>>`
  append below suits ephemeral builders; on a persistent runner, set the property once.
- **Seen on:** Kotlin 2.4.10, Gradle 8.14.5, Xcode 26.5, Codemagic `mac_mini_m2` (12 GB).

```yaml
- name: Size the Gradle JVM for the builder
  script: |
    mkdir -p "$HOME/.gradle"
    cat >> "$HOME/.gradle/gradle.properties" <<'EOF'
    org.gradle.jvmargs=-Xmx7g -XX:MaxMetaspaceSize=768m -Dfile.encoding=UTF-8
    kotlin.daemon.jvmargs=-Xmx1g
    org.gradle.parallel=false
    EOF
```

- Codemagic: `ios_signing` fails ("No matching profiles found") when no profile exists; use
  `fetch-signing-files --create`. A YAML merge key replaces `vars` whole. `BUILD_NUMBER` is
  Codemagic's own build index, not yours.

See also: kotlin-tooling-native-build-performance in Kotlin/kotlin-agent-skills (keeping `~/.konan` warm in CI, caching and target choices for faster Native builds).

## 4. If Swift implements a Kotlin `suspend fun`: complete the handler on every path

- **Symptom:** the Kotlin caller waits forever, or a soft failure becomes a Kotlin exception.
- **Cause:** Kotlin exports `suspend fun token(): String?` to Swift as
  `token(completionHandler: (String?, Error?) -> Void)`. The coroutine resumes only when Swift
  calls the handler. A method call on a nil SDK instance never runs its callback. A non-nil `Error`
  resumes the coroutine with an exception.
- **Fix:** call the handler exactly once on every path. For optional data, complete with
  `(nil, nil)` and let Kotlin decide.
- **Seen on:** Kotlin 2.4.10, Firebase iOS SDK 11.x.

```swift
func token(completionHandler: @escaping (String?, Error?) -> Void) {
    guard DCAppAttestService.shared.isSupported else { completionHandler(nil, nil); return }
    AppCheck.appCheck().token(forcingRefresh: false) { t, _ in completionHandler(t?.token, nil) }
}
```

- Firebase App Check: `AppAttestProvider(app:)` never returns nil. Pick the provider by
  `DCAppAttestService.shared.isSupported` (false on simulators); debug provider only in DEBUG.

## 5. If the binary supports iPad or links privacy-sensitive frameworks: App Store Connect checks the host config

- **Symptom:** the upload or processing fails with ITMS-90476 or ITMS-90683.
- **Cause:** ITMS-90476: a binary with `TARGETED_DEVICE_FAMILY` "1,2" must declare a launch
  screen, and a `UILaunchStoryboardName` that names a missing storyboard does not count.
  ITMS-90683: a linked framework references privacy-sensitive APIs (a WebRTC framework references
  the camera), so Apple requires the usage string (`NSCameraUsageDescription`) even if the app
  never calls them.
- **Fix:** use a `UILaunchScreen` dictionary with a colour asset (no storyboard). Add a true
  purpose string for every API that your frameworks reference. Build numbers after a rejection:
  see kmp-store-release rule 3.
- **Seen on:** App Store Connect, iOS deployment target 18.0.

## 6. If you use XcodeGen: the `entitlements:` block writes one file into every configuration

- **Symptom:** a Release archive signed with `aps-environment = development` gets sandbox APNs
  tokens, and production pushes to them fail with no error in the app.
- **Cause:** the `entitlements:` block generates one file for all configurations.
- **Fix:** keep two files and set `CODE_SIGN_ENTITLEMENTS` under `settings.configs.Debug` and
  `settings.configs.Release`. Put every entitlement that must ship (associated domains, App
  Attest environment) into both.
- **Seen on:** xcodegen, iOS deployment target 18.0.

## 7. If you use XcodeGen: the generated project, source globs and schemes

- **Symptom:** CI builds a stale project; the build creates thousands of duplicate resource-copy
  commands; the simulator resolves no StoreKit products.
- **Cause:** CI that does not run `xcodegen generate` builds the committed `.xcodeproj`. A source
  path that contains a build folder globs its output into the target. Without a `schemes:` block,
  xcodegen's default scheme has no StoreKit configuration.
- **Fix:** run xcodegen in CI, or regenerate and commit the project with every `project.yml`
  change (expect UUID churn). Exclude build folders. Declare the scheme, StoreKit file on Debug only.
- **Seen on:** xcodegen (project `objectVersion` 77), Xcode 26.5 on CI.

```yaml
schemes:
  App: { build: { targets: { App: all } }, run: { config: Debug, storeKitConfiguration: App.storekit } }
targets:
  App:
    sources: [{ path: App, excludes: ["build", "**/build"] }]
    dependencies: [{ sdk: libsqlite3.tbd }]          # SQLDelight native driver
```

## 8. A throw inside a Compose effect ends the process, on Android and on iOS

- **Symptom:** the app closes when an effect runs a call that throws. It can look iOS-only when
  the throwing call depends on iOS timing.
- **Cause:** Compose does not catch exceptions from `LaunchedEffect` or `DisposableEffect`. They
  are uncaught: the JVM default handler ends the Android process, and Kotlin/Native terminates the
  iOS process. Example: `FocusRequester.requestFocus()` throws while its node is not attached yet.
- **Fix:** catch what can throw inside effects, and retry the calls that depend on layout. Plain
  `runCatching` is fine for synchronous calls only. It also catches `CancellationException`, so
  around suspend calls catch and rethrow cancellation first (see kmp-ktor-networking rule 3).
  Focus timing: see kmp-sheets-keyboard.
- **Seen on:** Compose Multiplatform 1.11.1, Kotlin 2.4.10.

## 9. If you support iPad multitasking or Home-screen quick actions: UIKit scene and launch delivery

- **Symptom:** on iPad the window ignores Stage Manager and Split View; a quick action runs twice
  on cold launch.
- **Cause:** a window from `UIScreen.main.bounds` is fixed to the screen. UIKit delivers a
  cold-launch quick action in `launchOptions`, then calls `performActionFor` too if
  `didFinishLaunching` returns true.
- **Fix:** build the window in a `SceneDelegate` with `UIWindow(windowScene:)`. Keep
  `UIApplicationSupportsMultipleScenes` false while app state is process-global. Handle the
  launch quick action in `didFinishLaunching` and return false. If you route by hand, links
  that arrive before the Compose root exists: see kmp-app-architecture rule 6.
- **Seen on:** iOS 18 and 26, Compose Multiplatform 1.7.3 and 1.11.1.

## 10. If sign-in state lives in the Keychain: iOS keeps it after uninstall, Android only through Auto Backup

- **Symptom:** iOS, and Android when Auto Backup is on. After an uninstall and reinstall, onboarding runs again, but the app is
  signed in to the old account with its server state (a spent trial, old progress). The same
  steps on Android with `allowBackup="false"` give a new account.
- **Cause:** deleting an iOS app removes its container and `UserDefaults`, not its Keychain items.
  Firebase Auth on iOS stores the signed-in user (anonymous too) in the Keychain, so `currentUser`
  is back on the first launch. Android deletes app data with the app, unless Auto Backup
  (`android:allowBackup`, on by default) restores it at install.
- **Fix:** on iOS, empty prefs do not mean a new user. For a fresh test account, call
  `Auth.auth().signOut()` or run `xcrun simctl keychain <udid> reset`; uninstall is not enough. If
  a reinstall must start a new account, decide that in code. After an app transfer, the Keychain access group
  changes, so the old items are unreadable and anonymous users get new ids.
- **Seen on:** Firebase iOS SDK 11.x, iOS 26 Simulator and an iPhone, Android with `allowBackup="false"`.

## Moved to other skills

- Build platform objects outside the `ComposeUIViewController` content lambda: see kmp-app-architecture, rule 1.
- commonTest per target, commas in backticked test names: see kmp-testing-strategy.
- Compose resources on iOS system surfaces (Info.plist strings, push keys): see kmp-strings-localization.
- Launch screen to Compose handoff: see kmp-compose-visuals. `iosX64` removal: see kmp-compose-upgrade.

## Checklist

- [ ] Swift to Kotlin payloads are JSON strings or primitives; every drop logs.
- [ ] Each Swift `completionHandler` for a Kotlin `suspend fun` runs once on every path.
- [ ] The iOS test link gets what Xcode gives the app link.
- [ ] CI sets `org.gradle.jvmargs` in `~/.gradle/gradle.properties`, pins Xcode, caches `~/.konan`.
- [ ] Universal binary has `UILaunchScreen`; every referenced privacy API has a purpose string.
- [ ] Debug and Release entitlements are separate, complete files.
- [ ] xcodegen output is regenerated or built in CI; build folders excluded; scheme declared.
- [ ] No unguarded throwing call inside an effect, on either platform.
- [ ] iPad window comes from a `SceneDelegate`; a cold-launch quick action is handled once.
- [ ] "New user" logic does not assume an iOS reinstall clears the Keychain sign-in.
