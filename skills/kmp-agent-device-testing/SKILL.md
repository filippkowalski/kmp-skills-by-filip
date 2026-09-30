---
name: kmp-agent-device-testing
description: Drive and verify a Kotlin Multiplatform + Compose Multiplatform app on the iOS Simulator with the AXe CLI and on Android with adb and uiautomator, using one set of Compose testTags. Use when an agent or script must find, tap, type or screenshot on a simulator, emulator or device; when testTags do not show up as Android resource-ids or iOS accessibility ids; when axe or uiautomator finds nothing; or when simctl, Gradle install or adb input behave unexpectedly. Triggers - testTag, testTagsAsResourceId, resource-id, AXUniqueId, accessibility identifier, AXe, axe tap, describe-ui, uiautomator dump, adb input, simctl, installDebug, pm clear, set-app-locales, haptics under adb.
---

# Agent device testing for KMP + Compose Multiplatform

Tag interactive elements once in `commonMain` with `Modifier.testTag("snake_case_id")`. AXe reads
the iOS accessibility tree, `uiautomator` reads Android resource-ids. Most expensive traps first.

## 1. Android: a testTag is not a resource-id until an ancestor opts in

**Symptom:** hundreds of `testTag` calls, and `uiautomator dump` shows no `resource-id` for any of them.
**Cause:** Compose writes a tag into the Android accessibility tree only under an ancestor with
`testTagsAsResourceId = true`. An app composable that returns early on some branch (for example
onboarding) also misses those screens if the flag sits inside it.
**Fix:** one `expect`, identity on iOS, the flag on Android, applied at the activity root.

```kotlin
// commonMain
expect fun Modifier.mapTestTagsToAccessibilityIds(): Modifier
// androidMain
@OptIn(ExperimentalComposeUiApi::class)
actual fun Modifier.mapTestTagsToAccessibilityIds(): Modifier = semantics { testTagsAsResourceId = true }
// iosMain
actual fun Modifier.mapTestTagsToAccessibilityIds(): Modifier = this

// MainActivity.setContent (inside the Box, use this@MainActivity for a Context)
Box(Modifier.fillMaxSize().mapTestTagsToAccessibilityIds()) { App(/* ... */) }
```

**Seen on:** CMP 1.11.1, Kotlin 2.4.10.

## 2. Android: every sheet, dialog, menu and popup needs the flag again

**Symptom:** the screen's tags are resource-ids, but the buttons in a bottom sheet or dialog are not.
**Cause:** `ModalBottomSheet`, `Dialog`/`AlertDialog`, `DropdownMenu` and `Popup` draw in their own
window. Their semantics tree starts there, not under the activity root.
**Fix:** apply the mapping on the first node inside each window.

```kotlin
ModalBottomSheet(onDismissRequest = onDismiss) {
    Column(Modifier.mapTestTagsToAccessibilityIds().testTag("close_sheet")) { /* ... */ }
}
AlertDialog(modifier = Modifier.mapTestTagsToAccessibilityIds(), /* ... */)
DropdownMenu(expanded, onDismiss, modifier = Modifier.mapTestTagsToAccessibilityIds()) { /* ... */ }
```

After adding one, grep for `ModalBottomSheet`, `Dialog`, `DropdownMenu` and `Popup(`.
**Seen on:** CMP 1.11.1 (Material 3).

## 3. iOS: check that tags reach `AXUniqueId` before you select by id

**Symptom:** `axe tap --id <tag>` matches nothing; `axe describe-ui` shows labels but no id.
**Cause:** `--id` matches only `AXUniqueId`. A tag that does not reach that field on your CMP
version leaves `--id` nothing to match, and AXe reports no error for it.
**Fix:** check one known tag on the screen before writing any `--id` selector.

```sh
axe describe-ui --udid "$U" | grep -c -F '"save_button"'   # 0: the tag is not in the tree
```

If the count is 0, select by `--label` (rule 4). There is no switch to set: CMP 1.11.1 has no
`AccessibilitySyncOptions`, so delete any `configure { accessibilitySyncOptions = ... }` left from 1.7.
**Seen on:** CMP 1.11.1, iOS 26.5 Simulator, AXe 1.4.0 (ids absent).

## 4. If you drive the iOS Simulator with AXe: exact flags, exact labels, points

**Symptom:** a tap does nothing, or fails with "Either provide both -x/-y".
**Cause and fix (AXe 1.4.0):**
- Every command needs `--udid`.
- `--id` matches `AXUniqueId`, `--label` matches `AXLabel`: two dashes. Coordinate taps take one:
  `axe tap -x 90 -y 137 --udid "$U"`. `--x 90` fails.
- A merging `clickable` that sets `contentDescription = text` and also draws `Text(text)` reports a
  joined label: `"Continue, Continue"`. Typographic apostrophes stay (`Let’s`). Copy labels from
  `describe-ui` verbatim.
- Coordinates are points. On a 3x device, divide screenshot pixels by 3.
- `axe type` sends US-keyboard characters only. No accented letters.
- A default `axe swipe` is fast, so a screen recording shows the scroll jump. For a visible
  scroll use `--duration 2.5 --delta 1`.

**Poll; never sleep a fixed time.** Animations and cold starts show the target late, and a tap
before it exists does nothing, with no error.

```sh
wait_for_label() {  # $1 = exact AXLabel; gives up after 15 s
  for _ in $(seq 1 30); do axe describe-ui --udid "$U" | grep -qF "\"$1\"" && return 0; sleep 0.5; done
  echo "timeout waiting for $1" >&2; return 1
}
wait_for_label "Continue, Continue" && axe tap --label "Continue, Continue" --udid "$U"
```

**Seen on:** AXe 1.4.0, CMP 1.11.1.

## 5. iOS Simulator state that survives what you expect to reset it

- **App data:** there is no `pm clear`. `simctl uninstall` removes data and bundle, so reinstall
  from the build output (or copy the `.app` out of the device container first).
- **Keychain:** uninstall keeps it, so a stored sign-in comes back. Run `xcrun simctl keychain <udid>
  reset` for a fresh account (see kmp-ios-build rule 10).
- **Software keyboard:** it appears only until a hardware keyboard types. After `axe type` it stays
  hidden. A check of keyboard behaviour needs a freshly created simulator.
- **Reduce Motion:** writing `ReduceMotionEnabled -int 0` back with `simctl spawn ... defaults write
  com.apple.Accessibility` does not turn it off. Animations still snap. Reboot the simulator.

**Seen on:** iOS 26.4 and 26.5 Simulators.

## 6. If the Mac is also running emulators or heavy builds, CoreSimulator waits forever

**Symptom:** `simctl install`, `launch` and `listapps` hang for 10+ minutes; the screen is black.
**Cause:** with host load in the hundreds, CoreSimulator services starve. Calls wait, never fail.
**Fix:** run `uptime` first. Above about 200, run compiles and unit tests instead and retry the
simulator later. Kill your own hung `simctl` processes.
**Seen on:** macOS with iOS 26.5 Simulator.

## 7. Build a KMP app for a simulator by UDID

**Symptom:** `xcodebuild` fails with "Unknown iOS simulator arch: 'x86_64'".
**Cause:** the generic simulator destination asks for x86_64; the Kotlin framework has no x86_64
slice once `iosX64` is gone (CMP 1.11 does not publish it).
**Fix:** `-destination 'id=<udid>'`; for Release add `ARCHS=arm64 ONLY_ACTIVE_ARCH=YES`.
**Seen on:** CMP 1.11.1, Kotlin 2.4.10, Xcode 26.

## 8. If more than one device is attached, Gradle `installDebug` installs on all of them

**Symptom:** the APK lands on a device you did not mean. With one phone attached over USB and
Wi-Fi at once, it fails with "device not found".
**Cause:** the install task targets every device `adb devices` lists and picks its own handle.
**Fix:** `./gradlew :app:assembleDebug`, then
`adb -s "$S" install -r app/build/outputs/apk/debug/app-debug.apk`.

**Seen on:** AGP 8.7.3 and 8.13.2.

## 9. If debug and release share one application id, `pm clear` hits whichever is installed

**Symptom:** a `pm clear` or `uninstall` meant for the debug build removes a release install and
its data.
**Cause:** one application id is one package slot on the device. The signing keys differ, so
installing one build over the other also needs an uninstall.
**Fix:** check `adb -s "$S" shell dumpsys package <appId> | grep versionName` before wiping. See
kmp-store-release for what a debug `applicationIdSuffix` costs.

## 10. adb recipes that work with Compose

```sh
# Find a tagged element; resource-id is the raw tag. Tap the centre, in pixels.
adb -s "$S" shell uiautomator dump /sdcard/ui.xml >/dev/null
adb -s "$S" exec-out cat /sdcard/ui.xml \
  | grep -o 'resource-id="save_button"[^>]*bounds="[^"]*"' | grep -o 'bounds="[^"]*"'
adb -s "$S" shell input tap 540 2062          # centre of [84,1990][996,2134]
# Press and hold
adb -s "$S" shell input motionevent DOWN 540 2062; sleep 1.5
adb -s "$S" shell input motionevent UP 540 2062
# Launch. Do not use monkey: it can open launcher search instead.
adb -s "$S" shell am start -n <appId>/.MainActivity
# The app's own log only
adb -s "$S" logcat --pid=$(adb -s "$S" shell pidof <appId>)
# One capture cannot show a loop: compare two, a second apart
adb -s "$S" exec-out screencap -p > a.png; sleep 1; adb -s "$S" exec-out screencap -p > b.png
```

See also: android-cli in android/skills (`android layout` returns the screen as JSON with bounds, an alternative to `uiautomator dump`).

## 11. If you verify haptics: injected input fires none

**Symptom:** `dumpsys vibrator_manager` stays empty after `adb shell input tap` on a control whose
haptic call ran.
**Cause:** adb-injected input does not reach the vibrator through `View.performHapticFeedback`.
**Fix:** verify the visual state through adb; verify haptics by hand on a device.
**Seen on:** Android 17, physical device.

## 12. If you verify translations on Android: set a per-app locale

```sh
adb -s "$S" shell cmd locale set-app-locales <appId> --locales pl
adb -s "$S" shell am force-stop <appId>       # then launch again
adb -s "$S" shell cmd locale set-app-locales <appId> --locales ""   # reset
```

Needs Android 13 or later. CMP resources follow it. See kmp-strings-localization: plural
categories to exercise per language.
**Seen on:** Android 17, CMP 1.7.3 and 1.11.1.

## 13. Emulator traps

- A stale global proxy kills all network, auth included: `adb shell settings put global http_proxy :0`.
- A nearly full `/data` fails installs: `adb shell pm trim-caches 8G`.
- "System UI isn't responding" under host CPU load is starvation, not the app. Tap Wait.
- A killed emulator leaves `*.lock` files in its `.avd` folder; the next launch fails with
  "Running multiple emulators with the same AVD". Delete the locks.
- `-no-audio` turns off audio in and out; test any audio feature on a physical device.
- `adb wait-for-device` can block a shell forever. Poll `adb devices` in a short loop.
**Seen on:** Android emulator, API 35 image.

## Checklist

- [ ] `testTagsAsResourceId` at the activity root and inside every sheet, dialog, menu and popup
- [ ] One known tag counted in `describe-ui` before any `--id` selector; `--label` if it is 0
- [ ] AXe: `--udid` on every call, `-x/-y` for points, labels copied verbatim, polling not sleeps
- [ ] `uptime` checked before simulator work
- [ ] `assembleDebug` + `adb -s <serial> install -r`; never `installDebug` with two devices
- [ ] `versionName` checked before `pm clear` when debug and release share an id
- [ ] Haptics checked by hand; screenshot, recording or log saved with device, OS and build
