---
name: kmp-compose-upgrade
description: Upgrade diary for a Kotlin Multiplatform + Compose Multiplatform app moving from CMP 1.7 to 1.11, Kotlin 2.1 to 2.4, AGP 8 to 9 and kotlinx-datetime 0.6 to 0.7, with a known-good version set. Use when upgrading Compose Multiplatform, Kotlin, AGP, kotlinx-datetime, Haze or the AndroidX lifecycle in those ranges, or when a bump breaks one platform only. Triggers - Compose Multiplatform upgrade, CMP 1.11, CMP 1.8, Kotlin 2.4, AGP 9, com.android.kotlin.multiplatform.library, "not compatible with the org.jetbrains.kotlin.multiplatform plugin", module split, compileSdk 36, Haze ShaderBrush crash, HazeProgressive, kotlinx-datetime 0.7, Clock.System unresolved, kotlin.time.Clock, iosX64 missing, AccessibilitySyncOptions, platform() unresolved, lifecycle-runtime-compose 2.11.
---

# Compose Multiplatform upgrade (CMP 1.7 to 1.11, Kotlin 2.1 to 2.4, AGP 8 to 9)

What breaks on the way from CMP 1.7.3 / Kotlin 2.1.21 to CMP 1.11.1 / Kotlin 2.4.10, and a
version set that builds and ships on both platforms. Two habits make the jump safe:

- Move Kotlin, CMP and every library pinned to them in one change. A library that lags holds the
  whole set back until it has a line for the new toolchain.
- Declare the versions you depend on, even when a transitive dependency brings them. Then the
  numbers in the build file are the numbers that ship, and both platforms resolve the same major.

## 1. If you use kotlinx-datetime: 0.7 moves `Clock` and `Instant` to `kotlin.time`

- **Symptom:** `Clock.System` stops compiling for iOS while Android still builds.
- **Cause:** `compose.material3` pulls kotlinx-datetime 0.7.x for its date pickers, so platforms
  can resolve different majors. 0.7 moved `Instant` and `Clock` to `kotlin.time`. The typealiases
  0.7.1 leaves behind expose no nested classifier, so `Clock.System` no longer resolves.
- **Fix:** declare kotlinx-datetime 0.7.1 yourself. Import `Clock` and `Instant` from
  `kotlin.time`. Keep `TimeZone`, `LocalDate` and `todayIn` from `kotlinx.datetime`.
- **Seen on:** kotlinx-datetime 0.6.1 to 0.7.1, CMP 1.11.1, Kotlin 2.4.10.

```kotlin
import kotlin.time.Clock      // was kotlinx.datetime.Clock
import kotlin.time.Instant    // was kotlinx.datetime.Instant
```

## 2. If you move to AGP 9: the KMP plugin no longer shares a module with an Android plugin

- **Symptom:** configuration fails with "The 'com.android.library' (or 'com.android.application')
  plugin is not compatible with the 'org.jetbrains.kotlin.multiplatform' plugin since AGP 9.0."
- **Cause:** by default AGP 9 refuses both plugins in one subproject. A single `composeApp` module
  that applies `kotlin.multiplatform` and `com.android.application` is exactly that. The check
  runs with AGP's built-in Kotlin, which is on by default.
- **Fix:** split the module: shared code in a KMP library module with the
  `com.android.kotlin.multiplatform.library` plugin, a thin app module with
  `com.android.application`. Short-term options: stay on AGP 8.13.x (the library plugin already
  exists there), or set both `android.builtInKotlin=false` and `android.newDsl=false` in
  `gradle.properties`. The AGP 9.2.1 error names that pair as a temporary bypass; a later AGP may
  remove it, so still plan the split.
- Libraries can force AGP 9 first: JetBrains `lifecycle-runtime-compose` 2.11.0 needs AGP 9.1
  and compileSdk 37. On AGP 8 stay on 2.10.0 (CMP 1.11.1 asks for 2.9.6; 2.10.0 resolves).
- **Seen on:** AGP 8.13.2 and 9.2.1 (plugin, error text and bypass properties read from the AGP
  jars), CMP 1.11.1.

See also: kotlin-tooling-agp9-migration in Kotlin/kotlin-agent-skills (the step-by-step module split and plugin checks).

## 3. If you use Haze 1.x and move to CMP 1.11: styles that reach `ShaderBrush.createShader` crash

- **Symptom:** a crash on iOS (Skia) for a progressive blur, a brush tint or a mask. The JVM
  reports `NoSuchMethodError`; Kotlin/Native shows type confusion instead.
- **Cause:** CMP 1.11 changed the return type of `ShaderBrush.createShader`. Haze fixed its side
  only in 2.0.0-alpha01 (Haze issue #880). 1.7.2 is the last stable 1.x line.
- **Fix:** on Haze 1.7.2 use `HazeMaterials` or plain colour tints only, and write that limit next
  to the dependency. For the rest, move to Haze 2.x (separate `haze-blur` module, `blurEffect {}`).
- **Seen on:** Haze 1.7.2, CMP 1.11.1, Kotlin 2.4.10.

```kotlin
Modifier.hazeEffect(hazeState, HazeMaterials.thin(Color.White))   // safe on Haze 1.7.2 + CMP 1.11
// Not on 1.x with CMP 1.11: HazeProgressive, Brush tints, masks.
```

## 4. If you move to CMP 1.11: its AndroidX chain needs compileSdk 36

- **Symptom:** Android compilation rejects the AndroidX artifacts when compileSdk is 35.
- **Cause:** CMP 1.11 brings activity 1.12.2, navigationevent-compose 1.0.1 and lifecycle 2.10,
  which require compileSdk 36.
- **Fix:** raise compileSdk to 36. `targetSdk` is a separate decision: raising compileSdk turns on
  no new runtime behaviour.
- **Seen on:** CMP 1.11.1, AGP 8.13.2, activity-compose 1.12.2.

## 5. If you move to Kotlin 2.4 and import a BOM in a source set: bare `platform()` stops resolving

- **Symptom:** `implementation(platform("...bom..."))` inside `androidMain.dependencies { }` is
  unresolved after the Kotlin 2.4 bump.
- **Cause:** the KMP source-set dependency handler has no `platform()`. The bare call only worked
  through the enclosing script scope.
- **Fix:** `implementation(project.dependencies.platform("com.google.firebase:firebase-bom:33.7.0"))`.
- **Seen on:** Kotlin 2.4.10, Gradle 8.14.5.

## 6. If you move to CMP 1.11 and declare `iosX64`: the target is no longer published

- **Symptom:** dependency resolution fails for the `iosX64` target after the bump.
- **Cause:** CMP 1.11.1 `runtime`, `ui` and `foundation` ship `iosArm64` and `iosSimulatorArm64`
  only; on 1.7.3 `iosX64` still resolved.
- **Fix:** remove `iosX64()`. Intel simulators are not supported. Check that every library you
  add publishes the two remaining targets before you pick its version.
- **Seen on:** CMP 1.7.3 to 1.11.1 in one step, Kotlin 2.4.10.

## 7. If you move from CMP 1.7 to a later version: `AccessibilitySyncOptions` is gone

- **Symptom:** `ComposeUIViewController(configure = { accessibilitySyncOptions = ... })` no longer
  compiles.
- **Cause:** CMP 1.11.1 builds the iOS accessibility tree on demand and has no
  `AccessibilitySyncOptions`; 1.7.3 had it. There is no switch.
- **Fix:** delete the option. To check that test tags reach automation tools, see
  kmp-agent-device-testing.
- **Seen on:** CMP 1.7.3 to 1.11.1 in one step (the 1.11.1 `ui` klib has no such type).

## Owned elsewhere

- Ktor modules move as one set: see kmp-ktor-networking.
- Kotlin/Native rejects commas in backticked test names: see kmp-testing-strategy.
- Skia `Shader` vs Compose `Shader` (`asComposeShader()`), system bar icons, Coil caching: see
  kmp-compose-visuals. Draw-phase animation reads: see kmp-compose-motion.
- purchases-kmp 3.x and its iOS test link: see kmp-revenuecat and kmp-ios-build.
- `getString` deadlock, Compose resources: see kmp-strings-localization.
- `expect class` constructors in commonTest: see kmp-testing-strategy.
- `ModalBottomSheet` sizing and scrolling: see kmp-sheets-keyboard.
- State holder, navigation without a library: see kmp-app-architecture.

## Known-good version set (a set that shipped together in 2026-09)

These are working minimums, not the latest versions. The set builds and ships on Android and iOS
together. Rows marked optional matter only if you use them.

| Component | Version | Note |
|---|---|---|
| Kotlin (multiplatform, compose, serialization plugins) | 2.4.10 | |
| Compose Multiplatform | 1.11.1 | iOS targets: iosArm64, iosSimulatorArm64 |
| AGP / Gradle | 8.13.2 / 8.14.5 | AGP 9 needs the module split or the bypass (rule 2) |
| compileSdk / targetSdk / minSdk / JVM target | 36 / 36 / 26 / 17 | |
| kotlinx-coroutines / serialization-json / datetime | 1.10.1 / 1.7.3 / 0.7.1 | declare datetime |
| Ktor (every module) | 3.1.0 | |
| SQLDelight | 2.3.2 | optional |
| Coil 3 (+ network-ktor3, network-cache-control) | 3.5.0 | optional; built against CMP 1.11.1 |
| JetBrains lifecycle-runtime-compose | 2.10.0 | 2.11.0 needs AGP 9.1 |
| androidx activity-compose / lifecycle | 1.12.2 / 2.10.0 | |
| multiplatform-settings | 1.3.0 | optional |
| Haze + haze-materials | 1.7.2 | optional; colour tints only |
| purchases-kmp-core | 3.5.0 | optional; needs Kotlin 2.3+ |

The Gradle plugin aliases (`compose.runtime`, `compose.material3`, ...) still resolve on 1.11.1.

## Checklist

- [ ] Kotlin, CMP and every library pinned to them move in one change; every version is listed.
- [ ] kotlinx-datetime declared; `Clock` and `Instant` imported from `kotlin.time`.
- [ ] AGP 9: shared code in a `com.android.kotlin.multiplatform.library` module, app module thin.
- [ ] Haze 1.x on CMP 1.11: no progressive blur, brush tint or mask.
- [ ] compileSdk 36; targetSdk raised on purpose, not by the toolchain.
- [ ] `project.dependencies.platform()` inside KMP source sets.
- [ ] No `iosX64`; no `AccessibilitySyncOptions`.
- [ ] Both Android and iOS compile and run tests after the bump.
