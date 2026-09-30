---
name: kmp-compose-motion
description: Motion in Compose Multiplatform apps on iOS and Android. How to build animation with Animatable, animate*AsState, spring and tween, AnimatedVisibility, AnimatedContent, graphicsLayer, pointerInput and LazyList item animations so it behaves the same on both platforms, and the traps that make it recompose every frame, replay, skip, cut or drop work. Use when adding or reviewing an animation, a transition, press feedback, a swipe or drag, list item entrances, overscroll or reduced-motion support. Triggers on animation, motion, spring, tween, transition, Animatable, animateTo, animateFloatAsState, AnimatedVisibility, AnimatedContent, SizeTransform, graphicsLayer, drawBehind, press feedback, pointerInput, awaitFirstDown, VelocityTracker, reduced motion, MotionDurationScale, overscroll, OverscrollEffect, rubber band, lazy list item animation, animateItem.
---

# Motion in Compose Multiplatform

Compose APIs for good motion, and the traps that make it differ on Android and iOS. Most expensive first.

## Pairs with

Emil Kowalski's motion skills (MIT, https://github.com/emilkowalski/skills): `emil-design-eng`, `apple-design`,
`animate`, `review-animations`, `improve-animations`, `find-animation-opportunities` and `animation-vocabulary`.
They teach when and why to animate, with web code. Use them for the why, this skill for the Compose how.

## Common motion advice in Compose terms

| Advice | Compose form |
|---|---|
| Respond when the finger lands | the raw `pointerInput` down, not `onClick` (rule 6) |
| Interruptible, keeps momentum | `animateTo(target, spring(), initialVelocity)`: springs retarget from the current value and velocity; `tween` drops the velocity and restarts its duration |
| Arrive without bounce | `spring(dampingRatio = 1f)`; a ratio below 1 only when a gesture hands over speed |
| Reduced motion | your own flag; the system scale cuts, it does not reduce (rule 7) |

## 1. An animated value read in composition recomposes every frame

**Symptom:** a short animation rebuilds text layouts, semantics and brushes every frame; scrolling stutters.
**Cause:** reading `Animatable.value`, a `by animate*AsState` delegate or a scroll offset in a composable body
is a composition read. Value-taking modifiers (`background(color)`, `alpha(a)`, `scale(s)`) read it there too.
**Fix:** pass `State` or a `() -> Float` and read it inside `graphicsLayer {}`, `drawBehind {}`,
`drawWithContent {}` or a placement block. Remember the getter so the modifier stays equal.

```kotlin
@Composable fun rememberFill(target: Float): () -> Float {
    val a = remember { Animatable(0f) }
    LaunchedEffect(target) { a.animateTo(target, spring()) }
    return remember(a) { { a.value } }
}
val tint = animateColorAsState(if (selected) Selected else Idle)       // State, no `by`
Box(Modifier.graphicsLayer { scaleX = fill() }.drawBehind { drawRect(tint.value) })
Box(Modifier.graphicsLayer { alpha = (list.firstVisibleItemScrollOffset / 48.dp.toPx()).coerceIn(0f, 1f) })
```
A `by` delegate is fine when its only read sits inside such a lambda. **Seen on:** CMP 1.11.1.

## 2. Lazy lists and disposed screens replay or skip entrances

**Symptom:** an item replays its entrance after it scrolls out and back. The last rows appear without
animating. A screen replays its entrance every time its tab returns.
**Cause:** `LazyColumn` disposes items (and their `remember` state) that leave the viewport and composes only rows
that fit the current frame, which can be short (a keyboard inset still closing). Later rows see an entrance flag
already true, so an `Animatable` seeded from it starts at the end. A screen that leaves composition resets too.
**Fix:** decide "new" above the list, keyed by item id. A short fixed list that must all animate can be a
scrolling `Column`, which composes every row at once. A once-per-process entrance keeps its "played" flag
in process memory, not in `rememberSaveable` (see kmp-compose-visuals rule 10). An item that draws its
own entrance uses `animateItem(fadeInSpec = null)`.

```kotlin
var seen by remember { mutableStateOf<Set<String>?>(null) }             // ids on screen last frame
val before = seen
SideEffect { seen = items.mapTo(HashSet()) { it.id } }
LazyColumn { items(items, key = { it.id }) { item ->
    val appear = remember { !before.isNullOrEmpty() && item.id !in before }  // first load does not pop
    Row(item, appear, Modifier.animateItem(fadeInSpec = null))
} }
```
**Seen on:** CMP 1.11.1.

## 3. Animation state belongs to its call site, not to the item shown there

**Symptom:** the next page plays the last page's state backwards: a flipped panel turns back and shows the
new item's hidden face for a moment, or a bar drains before it fills.
**Cause:** `animate*AsState` and `remember { Animatable() }` belong to a position in the composition. New data at
that position keeps the old value, and the new target animates from it.
**Fix:** key the state to the item (`remember(item.id) { Animatable(0f) }`, `LaunchedEffect(item.id, flipped)`)
or wrap the item in `key(item.id) {}`, so a new item starts from its own initial value. **Seen on:** CMP 1.7.3.

## 4. Animatable has one owner: a new animation cancels the coroutine running the old one

**Symptom:** the user interrupts an animation and the work after `animateTo` never runs: a commit is dropped
and a busy flag stays true, so the controls stay disabled.
**Cause:** `Animatable` guards changes with a `MutatorMutex`. Any other `animateTo` or `snapTo` cancels the running
one, and the `CancellationException` ends the whole coroutine that called it.
**Fix:** reset flags in `finally`. While a committed animation owns the value, gesture callbacks leave it alone.

```kotlin
scope.launch { busy = true; try { offset.animateTo(exitX, tween(220)); commit() } finally { busy = false } }
onDragEnd = { if (!busy) scope.launch { offset.animateTo(0f, spring()) } }    // never steal it mid-commit
```
**Seen on:** CMP 1.7.3.

## 5. VelocityTracker needs positions in a frame that does not move with the drag

**Symptom:** flings never register; only a distance threshold ends a swipe.
**Cause:** `PointerInputChange.position` is local to the node that holds `pointerInput`. When a translation earlier
in the chain (`graphicsLayer { translationX }`, `offset`) moves that node with the finger, the local position
barely changes and the tracker reads almost zero.
**Fix:** feed the tracker the running sum of drag deltas.

```kotlin
var dragged = 0f
detectHorizontalDragGestures(onDragStart = { tracker.resetTracking(); dragged = offset.value }) { change, dx ->
    dragged += dx; tracker.addPosition(change.uptimeMillis, Offset(dragged, 0f))
    scope.launch { offset.snapTo(offset.value + dx) }
}
```
**Seen on:** CMP 1.7.3.

## 6. clickable is late for press feedback

**Symptom:** a control reacts when the finger lifts, or about 0.1 s after it lands inside a list. A haptic
arrives after the network call the click started.
**Cause:** `onClick` fires on release. With a scrollable parent, `clickable` delays its pressed interaction (what
`Indication` and `collectIsPressedAsState()` see): the tap timeout on Android (`ViewConfiguration.getTapTimeout()`),
150 ms on iOS (the `UIScrollView` content-touch delay). `clickable` also consumes the down, so a `pointerInput`
placed before it never sees an unconsumed down.
**Fix:** drive feedback from the raw down, placed before the click modifier. Values are starting points.

```kotlin
@Composable fun Modifier.pressScale(onDown: () -> Unit = {}): Modifier {
    val s = remember { Animatable(1f) }; val scope = rememberCoroutineScope()
    return graphicsLayer { scaleX = s.value; scaleY = s.value }.pointerInput(Unit) { awaitEachGesture {
        awaitFirstDown(requireUnconsumed = false); onDown()               // haptic goes here
        scope.launch { s.animateTo(0.97f, spring(dampingRatio = 1f, stiffness = 1200f)) }
        waitForUpOrCancellation()                                        // null when a scroll takes over
        scope.launch { s.animateTo(1f, spring(dampingRatio = 0.75f, stiffness = 400f)) }
    } }
}
```
A component that owns its taps can use `detectTapGestures(onPress = { ...; tryAwaitRelease() })`.
**Seen on:** CMP 1.7.3 and 1.11.1; the delays read from CMP 1.11.1 foundation.

## 7. The OS motion setting cuts Compose animations; reduced motion needs your own flag

**Symptom:** with Reduce Motion (iOS) or Remove animations (Android) on, a short fallback fade never plays and
content cuts. On CMP 1.7, iOS plays every spring anyway. Common code cannot ask which mode is on.
**Cause:** Compose animates under a `MotionDurationScale` in the recomposer's context. At 0, `Animatable`,
`animate*AsState`, `AnimatedVisibility` and transitions jump to the end on the first frame. Android sets it live
from `Settings.Global.ANIMATOR_DURATION_SCALE` (other values stretch or shrink durations, springs too). CMP 1.8+
on iOS sets 0 while Reduce Motion is on, read when the view joins a window and when the scene becomes active;
CMP 1.7 ignores it. Flings run at a fixed scale of 1. A `CompositionLocal` nobody provides returns its default.
**Fix:** observe the setting per platform, provide it at the root, and branch on it.

```kotlin
@Composable expect fun rememberReducedMotion(): Boolean
// android: Settings.Global.getFloat(resolver, ANIMATOR_DURATION_SCALE, 1f) == 0f, plus a ContentObserver
// ios: UIAccessibilityIsReduceMotionEnabled(), plus UIAccessibilityReduceMotionStatusDidChangeNotification
val LocalReducedMotion = staticCompositionLocalOf { false }
CompositionLocalProvider(LocalReducedMotion provides rememberReducedMotion()) { App() }
private object FullSpeed : MotionDurationScale { override val scaleFactor = 1f }
LaunchedEffect(shown) { withContext(FullSpeed) { alpha.animateTo(1f, tween(200)) } }  // a fade that must play
```
Under the flag, drop travel and scale. A fade that must still play needs an `Animatable` run at your own
scale, as foundation does for flings (`animate*AsState` takes no context).
**Seen on:** CMP 1.7.3 and 1.11.1 (androidx Compose UI 1.7 and 1.11 on Android).

## 8. Overscroll differs per platform; one custom OverscrollEffect makes it match

**Symptom:** the same list rubber-bands on iOS and stretches on Android.
**Cause:** foundation picks the effect per platform: CMP on iOS installs a Cupertino rubber band and fling,
Android draws its stretch.
**Fix:** if both must feel the same, pass your own: `LazyColumn(overscrollEffect = remember { Band(scope) })`.
- `applyToScroll`: when the finger reverses with the band out, unwind it before `performScroll`, or the list
  scrolls while displaced. Grow the pull only from `NestedScrollSource.UserInput`.
- `applyToFling`: the leftover velocity, clamped (synthetic flings far outrun a thumb), is the `initialVelocity`
  of a critically damped spring home. Move content in the effect's `node` with `placeWithLayer` (no re-measure).
- iOS-like compression, a starting point: `d * (1 - 1 / (abs(pull) * 0.55f / d + 1))`, `d` = viewport.

In a sheet, overscroll fights drag-to-dismiss: see kmp-sheets-keyboard. **Seen on:** CMP 1.11.1.

## 9. AnimatedContent clips the size morph and reuses content that is still leaving

**Symptom:** during a switch the taller new content is sliced until the size animation ends. Going back
quickly shows stale data computed for the earlier state.
**Cause:** the default `SizeTransform` clips to the animating bounds. `AnimatedContent` keeps each state's content
composed while it exits and reuses it if the target returns first, so its `remember {}` keeps old values.
**Fix:** `SizeTransform(clip = false)` and let the container's shape bound the paint. Inside the content, key
`remember` on every input that can change while it is away: `remember(otherAnswer) { build(otherAnswer) }`.

```kotlin
AnimatedContent(mode, transitionSpec = {
    (fadeIn(tween(200, 60)) + scaleIn(tween(240, 60), 0.96f)) togetherWith fadeOut(tween(110)) using
        SizeTransform(clip = false) { _, _ -> tween(280) }
}) { m -> Content(m) }
```
**Seen on:** CMP 1.7.3 (clip) and 1.11.1 (reuse).

## Owned elsewhere

- See kmp-compose-visuals: splash handoff, `withFrameNanos` frame caps, once-per-launch flags, `AnimatedVisibility` in a `Popup`, haptics.
- See kmp-sheets-keyboard: sheet jitter; focus requests during an entrance on iOS; `AnimatedVisibility` exit content.

## Checklist

- [ ] No animated value read in a composable body; getters read in `graphicsLayer` or draw lambdas.
- [ ] Entrance decisions live above lazy lists and disposed screens; per-item state keyed to the item.
- [ ] Code after `animateTo` survives interruption; flags reset in `finally`.
- [ ] Velocity tracked in a frame that does not move; press feedback and haptics fire on the raw down.
- [ ] Reduced motion observed per platform, provided at the root, tested with the switch on both platforms.
- [ ] One overscroll feel, if the design needs it, through a custom `OverscrollEffect`.
- [ ] `AnimatedContent` size morph unclipped where content grows; `remember` inside it keyed.
