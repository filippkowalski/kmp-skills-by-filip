---
name: kmp-compose-visuals
description: Rendering and platform-visual behaviour in Compose Multiplatform apps on iOS and Android. Covers the Android splash-to-Compose handoff, runtime shaders (AGSL and SkSL), popup entrance jumps, LazyList scroll anchoring on live inserts, gradients on part of a text, haptics, edge-to-edge system bar icons, Coil 3 disk caching and Android adaptive icons. Use when holding the splash until the first screen is ready, writing a shader, animating a popup, keeping a live list pinned to new items, adding haptic feedback, fixing unreadable status bar icons, loading images whose content can change, replacing the app icon, or running something once per app launch. Triggers on installSplashScreen, setKeepOnScreenCondition, RuntimeShader, RuntimeEffect, ShaderBrush, Popup, PopupPositionProvider, scrollToItem, SpanStyle brush, HapticFeedbackType, enableEdgeToEdge, isAppearanceLightStatusBars, Coil, CacheControlCacheStrategy, adaptive-icon, rememberSaveable, warm start.
---

# Visual behaviour on iOS and Android in Compose Multiplatform

General traps in rendering and platform visuals. Most expensive first.
Animation state reads and lazy-list entrances: see kmp-compose-motion.

## 1. Hold the Android splash until the first screen is ready, and release it from composition

**Symptom:** the splash hangs until a timeout, or it cross-fades and dims before the first Compose
frame is ready.
**Cause:** `setKeepOnScreenCondition` holds the splash with an `OnPreDrawListener` that cancels draws.
A flag set in the draw phase never flips while draws are cancelled. The default exit is a cross-fade.
**Fix:** set the ready flag in composition (`SideEffect`), put a time cap on the condition, and call
`remove()` on exit when the first frame already matches the splash.

```kotlin
val splash = installSplashScreen()                              // before super.onCreate
val shownAt = SystemClock.uptimeMillis()
splash.setKeepOnScreenCondition { !firstScreenReady && SystemClock.uptimeMillis() - shownAt < 1_500 }
splash.setOnExitAnimationListener { it.remove() }               // cut, no cross-fade
// commonMain, in the first screen
val image = imageResource(Res.drawable.hero)                    // 1x1 placeholder until decoded
if (image.width > 1) SideEffect { firstScreenReady = true }     // composition, never draw
```

If you continue the splash with a Compose launch animation:
- The Android 12+ splash icon fits a 192dp circle on a 288dp canvas. Larger or animated art must be
  Compose. `provider.iconAnimationStartMillis` (wall clock) gives the frame of an `animation-list` icon
  at the handoff; the exit callback runs during a draw pass, so apply that value in the draw step.
- Keep the "played" flag in process memory, not in `rememberSaveable` (rule 10).
- Drive it with `withFrameNanos` and cap each step (about 34 ms), so a long first frame pauses the
  animation instead of skipping it.
- iOS `UILaunchScreen` with `UIImageRespectsSafeAreaInsets = false` centres the image on the full
  window. Centre the Compose continuation on the full window too.

**Seen on:** core-splashscreen 1.2.0, CMP 1.11.1, Android 12+, iOS 26.

## 2. If you use runtime shaders, the lifecycle differs on Android and iOS

**Symptom:** a shader animates on Android but freezes or fails on iOS, or crashes Android below 13.
**Cause:** Android `RuntimeShader` (AGSL) exists from API 33 and updates uniforms in place. Skia's
`RuntimeShaderBuilder.makeShader()` bakes uniforms into each shader. On CMP 1.11 a skia `Shader` is not
the Compose `Shader`. `RuntimeEffect.makeForShader` throws on a program Skia rejects.
**Fix:** one source string (AGSL and SkSL share `uniform shader`, `half4 main(float2)`, `eval` and common
math). Android code in a `@RequiresApi(33)` class with a still-image fallback. On iOS rebuild per frame,
convert, and wrap the compile.

```kotlin
@RequiresApi(33) private class Runtime(src: String) {        // androidMain: one brush for all frames
    val shader = RuntimeShader(src); val brush: Brush = ShaderBrush(shader)
    fun update(t: Float) = shader.setFloatUniform("t", t)
}
private val builder = runCatching { RuntimeShaderBuilder(RuntimeEffect.makeForShader(SRC)) }.getOrNull()
fun brush(t: Float): Brush? = builder?.let {                  // iosMain
    it.uniform("t", t); ShaderBrush(it.makeShader().asComposeShader())
}
```

**Seen on:** CMP 1.11.1, Android API 33+, iOS 26. Haze on CMP 1.11: see kmp-compose-upgrade.

## 3. AnimatedVisibility inside a Popup lands low, then jumps

**Symptom:** a popup placed above an anchor appears too low for one frame, then jumps into place.
**Cause:** the default `AnimatedVisibility` enter includes `expandIn`, so the content measures 0 on the
first frame. A `PopupPositionProvider` that places the popup by its measured height uses that 0.
**Fix:** animate scale and alpha in `graphicsLayer` on content that has its full size from the first
frame. Key it on the anchor so each new anchor replays the entrance.
**Seen on:** CMP 1.7.3 and 1.11.1.

## 4. firstVisibleItemIndex is stale after live inserts (sync, feed)

**Symptom:** a list that should stay pinned to its newest item stops following after a batch arrives.
**Cause:** a `LazyList` keeps its scroll anchor by key. Two items inserted in one frame move the anchor
from index 0 to 2 before your code reads it, so "was the user at the newest item" reads false. The next
key change can also cancel an `animateScrollToItem` halfway.
**Fix:** decide "pinned" when a scroll ends, and apply it with `scrollToItem`. Item placement animation
supplies the motion.

```kotlin
var pinned by remember { mutableStateOf(true) }
LaunchedEffect(listState) {
    snapshotFlow { listState.isScrollInProgress }.filter { !it }
        .collect { pinned = listState.firstVisibleItemIndex <= 1 }   // newest item at index 0
}
LaunchedEffect(items.size) { if (pinned) listState.scrollToItem(0) }
```

**Seen on:** CMP 1.11.1.

## 5. If you put a gradient on part of a text, the brush is sized to the whole paragraph

**Symptom:** a gradient on one word shows only a slice of the ramp, often the flat end color.
**Cause:** the span's shader gets the size of the whole paragraph, not the span.
**Fix:** measure the span in `onTextLayout` (`getBoundingBox` of its first and last character) and build
the brush between those corners. Keep the default brush when the span wraps. Write state only when the
box changes. **Seen on:** CMP 1.11.1, Android. iOS not checked.

## 6. Haptics: start with LocalHapticFeedback, and check that each type maps

**Symptom:** a tap gives no vibration on some devices or on one platform. A cue feels late.
**Cause:** `LocalHapticFeedback` maps `HapticFeedbackType` to `HapticFeedbackConstants` on Android and to
UIKit generators on iOS. Newer Android constants (API 30 gesture constants) can map to no effect on a
device, and the CMP 1.11.1 iOS mapping has no branch for `KeyboardTap`. A cue fired from state that
returns after async work lands late.
**Fix:**

```kotlin
val haptics = LocalHapticFeedback.current
Modifier.pointerInput(Unit) { detectTapGestures(onPress = { haptics.performHapticFeedback(HapticFeedbackType.LongPress) }) }
```

- Fire the cue from the gesture on press. Play an outcome cue (`Confirm`, `Reject`) when the result lands.
- Use types that map on both platforms (`LongPress`, `Confirm`, `Reject`, `SegmentTick`,
  `ToggleOn`/`ToggleOff`) and feel them on real devices.
- For a stronger pulse than the common API gives, use expect/actual: Android
  `VibrationEffect.createOneShot` (35 ms at full amplitude is a starting point) skipped when
  `Settings.System.HAPTIC_FEEDBACK_ENABLED` is 0, with `android.permission.VIBRATE` (normal, no prompt);
  iOS `UIImpactFeedbackGenerator` with a heavier style.
- adb-injected taps never vibrate: see kmp-agent-device-testing.

**Seen on:** CMP 1.11.1 `HapticFeedbackType`, Android `HapticFeedbackConstants` (API 26 to 36), iOS 26.

## 7. Status bar icons need edge-to-edge on every API level and one owner

**Symptom:** clock and battery icons are invisible on some Android versions or screens.
**Cause:** below API 35 the window is not edge-to-edge by default, and the theme paints an opaque bar
under the icons. The icon flags are window state that a recreated Activity resets. A `SideEffect` in a
skippable composable with a constant argument does not re-run after another call site changes them. On
iOS, `UIUserInterfaceStyle = Light` pins dark status bar text; no composable can change it.
**Fix:** call `enableEdgeToEdge()` on every API level. `ComponentActivity.enableEdgeToEdge()` sets
the icon colours itself, from the system dark mode by default (`SystemBarStyle.auto`), so set the
flags yourself only when the surface under the bars does not follow the system theme, and then from
one composable in a `SideEffect`. On iOS keep surfaces under the bar light, or override
`preferredStatusBarStyle` in the hosting view controller.

```kotlin
@Composable actual fun SystemBarIcons(dark: Boolean) {      // dark icons for light backgrounds
    val view = LocalView.current
    val window = (view.context as? Activity)?.window ?: return
    SideEffect { WindowCompat.getInsetsController(window, view).run {
        isAppearanceLightStatusBars = dark; isAppearanceLightNavigationBars = dark } }
}
```

**Seen on:** targetSdk 36, activity-compose 1.12.2. The below-35 path follows the platform contract and
was not run on a device.

See also: edge-to-edge in android/skills (Android insets, the `WindowCompat` vs `ComponentActivity` icon rule, navigation bar contrast).

## 8. If image content can change at the same URL, Coil 3 will not revalidate it by default

**Symptom:** a new image uploaded at the same URL never reaches devices.
**Cause:** Coil 3's default cache strategy serves a disk hit forever and never sends a conditional
request. `Cache-Control`/`ETag` support is a separate artifact.
**Fix:** add `coil-network-cache-control` and pass `CacheControlCacheStrategy`. Only the lambda
overload of `KtorNetworkFetcherFactory` takes a strategy; it also lets Coil use your Ktor client. Put the
disk cache in an app-owned caches directory through an `expect fun`.

```kotlin
@OptIn(ExperimentalCoilApi::class)                // install once with setSingletonImageLoaderFactory
fun imageLoader(ctx: PlatformContext, client: HttpClient) = ImageLoader.Builder(ctx)
    .components { add(KtorNetworkFetcherFactory(httpClient = { client }, cacheStrategy = { CacheControlCacheStrategy() })) }
    .diskCache { DiskCache.Builder().directory(imageCacheDir(ctx)).maxSizeBytes(64L shl 20).build() }.build()
```

**Seen on:** Coil 3.5.0, Ktor 3.1.0.

## 9. If you replace the launcher icon, change the XML too

**Symptom:** the new icon ships with the old background color, themed icon or shortcut circles. The
build does not fail.
**Cause:** adaptive icons take layers and colors from XML, not from the mipmap PNGs.
**Fix:** launchers show at most the inner 72dp of the 108dp foreground: keep corner details inside it
and fill the bleed by extending the art's edge. Draw the `monochrome` layer for Android 13+ themed icons
(a one-color flattening of detailed art is unreadable). Update `mipmap-anydpi-v26/ic_launcher.xml` and
`ic_launcher_round.xml`, the background drawable color, shortcut drawables that match it, and both
`android:icon` and `android:roundIcon`. Grep `res/` for the old hex color.
**Seen on:** Android 8+ adaptive icons, AGP 8.13.

## 10. If something must run once per app launch, rememberSaveable is the wrong lifetime

**Symptom:** a launch animation, intro or "app opened" event plays again when the user reopens an app
whose process never died. After a process death and a restore from Recents, it does not play at all.
**Cause:** Android restores saved instance state after a configuration change or a process death. When
the user dismisses the Activity (`finish()`, a swipe from Recents, Back on Android 11 and lower; 12+
moves a root Activity to the background instead), the state is dropped, and the next launch in the
same process gets the initial value. After a process death, the restored `rememberSaveable` flag is
already true, so nothing plays.
**Fix:** keep the flag in a top-level `var` (process lifetime, main thread) and seed state from it:
`var done by remember { mutableStateOf(launchPlayed) }` plus `SideEffect { launchPlayed = true }`.
Keep `rememberSaveable` for user state that must survive rotation. **Seen on:** CMP 1.11.1, Android 12+.

## Checklist

- [ ] Splash ready flag set in composition; condition time-capped; exit listener calls `remove()`.
- [ ] Once-per-launch flags in process memory, not `rememberSaveable`; launch animation steps capped.
- [ ] Shaders (if any): Android code in a `@RequiresApi(33)` class with a fallback; iOS shader rebuilt
      per frame, converted with `asComposeShader()`, compile wrapped.
- [ ] Popups animate in `graphicsLayer`, not with `AnimatedVisibility`'s default enter.
- [ ] Live lists decide "pinned" when a scroll ends and apply it with `scrollToItem`.
- [ ] Partial-text gradients sized to the span's box.
- [ ] Haptics via `LocalHapticFeedback`, fired on press, with types that map on both platforms.
- [ ] `enableEdgeToEdge()` on all API levels; one owner sets bar icon flags.
- [ ] Coil uses `CacheControlCacheStrategy` and an app-owned disk cache when URLs can be overwritten.
- [ ] Icon swap covers adaptive XML, colors, shortcuts, monochrome layer and manifest attributes.
