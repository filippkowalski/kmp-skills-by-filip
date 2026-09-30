---
name: kmp-app-architecture
description: App-layer rules for Kotlin Multiplatform + Compose Multiplatform apps, with or without ViewModels or a DI framework. Use when you wire the composition roots (MainActivity, MainViewController), choose between expect/actual, an injected interface and a Swift bridge, write state updates that cross a suspension, order a launch sequence, or route deep links and quick actions. Triggers - composition root, setContent, ComposeUIViewController, object graph, expect/actual, platform adapter, Swift-only SDK, SPM, state holder, stale response, race, account switch, data class copy, snapshot overwrite, LaunchedEffect cancelled, sibling launch, deep link, universal link, app link, quick action, route bus, configuration change, Activity recreation.
---

# KMP app layer: object graph, state updates, entry points

These rules hold whether state lives in a ViewModel or in a plain holder class. Each one answers
a behaviour of Compose, Kotlin/Native, kotlinx.coroutines or the Android and iOS entry points.

## 1. Composition can re-run its content lambda: build the object graph outside it

- **Symptom:** A second database driver, a second store SDK instance, a cache directory opened
  twice. The state holder is rebuilt and its state resets.
- **Cause:** The lambdas passed to `setContent { }` and `ComposeUIViewController { }` can run
  again. Objects constructed inside them are new on each run. If they are `remember` keys, every
  dependent `remember` is rebuilt too. On Android, `Activity.onCreate` also runs again on every
  configuration change (rule 7), so a driver built there leaks one copy per rotation.
- **Fix:** Build drivers and SDK clients once per process: in `Application.onCreate` or a lazy
  process singleton on Android, in a Kotlin `object` or the app delegate on iOS. The Activity and
  `MainViewController(...)` only pass them into the root. Build only Activity-bound pieces (a
  permission prompt) per Activity. Hoist function references used as `remember` keys out of the
  lambda too. Make every adapter a required parameter of the root composable, so a missing
  platform wiring is a compile error.

```kotlin
class App : Application() {
    val graph by lazy { AppGraph(this) }                 // one per process
}
class AppGraph(context: Context) {
    val store = SqlStore(DatabaseDriverFactory(context))  // owns a driver
    val purchases = StorePurchasesAdapter()               // owns an SDK
}
override fun onCreate(savedInstanceState: Bundle?) {      // MainActivity: wiring only
    super.onCreate(savedInstanceState)
    val graph = (application as App).graph
    setContent { AppRoot(store = graph.store, purchases = graph.purchases) }
}
```

- **Seen on:** Compose Multiplatform 1.7.3 and 1.11.1, Kotlin 2.1.21 and 2.4.10.

## 2. Kotlin/Native cannot call a Swift-only API: pick expect/actual, an interface, or a Swift bridge

- **Symptom:** commonMain needs an iOS API that exists only in Swift (no Objective-C interface),
  and no `actual` can import it. Or an `expect` declaration grows state, permissions and an
  Activity dependency, and tests cannot replace it.
- **Cause:** Kotlin/Native imports Objective-C only: headers through cinterop, and from Kotlin 2.4
  the Objective-C (Clang) modules of Swift packages through `swiftPMDependencies`. A Swift-only
  API stays invisible to Kotlin. `expect`/`actual` is resolved at compile time per target, so it
  cannot be swapped for a fake and cannot carry per-instance lifecycle.
- **Fix:** Choose per need.

| Need | Mechanism |
|---|---|
| Stateless platform fact or tiny function (debug flag, app version, DB driver factory) | `expect`/`actual` |
| Service with state, lifecycle or a permission (purchases, push token, biometrics, HealthKit, location) | interface in commonMain, platform class built in the composition root |
| Swift-only API (no Objective-C interface) | Kotlin interface in iosMain, Swift class implements it, passed into `MainViewController(...)` |

  On Android, hand an adapter a capability (a `suspend () -> Boolean` permission prompt owned by
  the Activity), not the Activity. For Swift implementations of `suspend` functions, see
  kmp-ios-build: Swift bridge callbacks.
- **Seen on:** Kotlin 2.1.21 and 2.4.10, iOS SDKs added through Swift Package Manager in Xcode.

See also: kotlin-tooling-cocoapods-spm-migration in Kotlin/kotlin-agent-skills (`swiftPMDependencies` setup and which Swift packages Kotlin can import).

## 3. Any state can change while a coroutine is suspended: check ownership after every suspension

- **Symptom:** A response for a signed-out or deleted account writes into the next account. An
  older read lands after a newer write and undoes it. A double tap starts two flows and the
  stale one wins.
- **Cause:** At each suspension point other coroutines run, on `Dispatchers.Main` too.
  Coroutines give no ordering between the responses of concurrent requests.
- **Fix:** Capture the owner (and a generation number for restartable actions) before the call.
  Compare after every suspension and drop the result when it no longer matches. Put reads and
  writes of one resource behind one `Mutex`, and check the owner inside the lock.

```kotlin
val owner = state.userId ?: return
val gen = ++startGeneration
fun stillOurs() = startGeneration == gen && state.userId == owner
scope.launch {
    val read = api.start(owner)            // suspends
    if (!stillOurs()) return@launch        // account changed or a newer start won
    state = state.withStarted(read)
}

private val lock = Mutex()
suspend fun refresh(owner: String) = lock.withLock {
    if (state.userId != owner) return@withLock
    val fresh = api.fetch(owner)
    if (state.userId == owner) state = state.copy(entitlement = fresh)
}
```

  Entitlement and purchase specifics: see kmp-revenuecat.
- **Seen on:** kotlinx-coroutines 1.10.1.

## 4. A copy taken before a suspension is a snapshot: merge results into live state

- **Symptom:** After a background sync, the items the user added meanwhile vanish. A sheet or
  paywall opened during a request closes by itself. A grant that arrived mid-save is lost.
- **Cause:** `state.copy(...)` captures every field at the moment it runs. A function that builds
  a whole new state from that copy, suspends, and returns it for assignment writes old values over
  everything that changed in between.
- **Fix:** Return focused result types from the data layer, never a whole state. Write one merge
  function per result that copies only the fields that read owns onto the current state, after
  the ownership check. Keep presentation flags (open sheets, paywall) out of any object that loads
  write back. With `StateFlow`, merge inside `update { }` from `it`, never from a captured copy.

```kotlin
internal fun AppState.withSync(read: SyncRead) = copy(
    items = read.items ?: items,
    lastSyncedAt = read.syncedAt,
    isSyncing = false,
)
_state.update { it.withSync(read) }   // or: state = state.withSync(read)
```

  Test it: block the request on a `CompletableDeferred`, change state, release, and assert both
  the change and the read's own fields survived.
- **Seen on:** Compose `mutableStateOf`, kotlinx-coroutines 1.10.1, Kotlin data classes.

## 5. Composition-scoped coroutines die with the screen: one job owns an ordered sequence

- **Symptom:** Leaving a screen mid-launch (tab switch, sheet, rotation) cancels a save. A
  "sequence done" signal fires before the last step. Cancelling the sequence leaves a request
  running.
- **Cause:** `LaunchedEffect` and `rememberCoroutineScope` coroutines are cancelled when their
  composable leaves composition. `otherScope.launch { }` inside a job is a child of `otherScope`,
  not of that job: the job completes without waiting, and cancelling it does not stop the work.
- **Fix:** Run work that must outlive a screen on the holder's (or ViewModel's) scope. Inside the
  owning job, call each step as a suspend function; never launch a step beside it. Expose the
  `Job` so a test can assert what it spans.

```kotlin
fun startLaunch() {
    if (launchJob?.isActive == true) return
    launchJob = scope.launch {        // holder scope, not a composable's
        flushPendingWrites()          // suspend, awaited
        refreshItems()                // suspend, awaited; not scope.launch { refreshItems() }
    }
}
```

- **Seen on:** kotlinx-coroutines 1.10.1, Compose Multiplatform 1.7.3 and 1.11.1.

## 6. If you route deep links or quick actions by hand (no navigation library): one route type, one parser, one slot

- **Symptom:** Platforms deliver external entries early and twice. A cold-start deep link is lost. A link replays after rotation. A quick action and
  the matching universal link open different places. A route cannot open a sheet.
- **Cause:** Android redelivers the launching Intent to a recreated Activity. On iOS a cold start
  can deliver a URL open and a `continue userActivity` for one link, and links arrive before the
  Compose root exists. Separate handlers per entry drift. State held in `remember` is unreachable
  from platform entry points.
- **Fix:**
  - A sealed `AppRoute` and one `parse(raw, source)` for https links and your custom scheme.
    Match exact path tokens: a prefix or bare-host claim also captures web pages (privacy, terms)
    that must open in the browser. A link that is yours but unknown resolves to Home.
  - Make each quick action's type a custom-scheme URL, so a shortcut is a deep link.
  - Park routes in one slot (last write wins), take it once, and apply it in one effect once the
    UI can navigate. Stamp the Android Intent with an extra after consuming it.
  - Put whatever a route opens in holder state, not in `remember`.
  - UIKit scene and launch-option delivery: see kmp-ios-build.

```kotlin
object AppRouteBus {
    var pending: PendingRoute? by mutableStateOf(null); private set
    fun offer(raw: String?, source: RouteSource): Boolean =
        AppRoutes.parse(raw, source)?.also { pending = it } != null
    fun take(): PendingRoute? = pending.also { pending = null }
}
```

- **Seen on:** Android `singleTask` activity with `onNewIntent`, iOS 18+, Compose Multiplatform
  1.7.3 and 1.11.1.

## 7. If state lives in a plain holder, not a ViewModel: it dies with the Activity

- **Symptom:** Rotation, dark mode or a system language change resets app state. A suspended
  permission request never resumes. The old holder's coroutines keep running.
- **Cause:** Without `android:configChanges`, a configuration change destroys and recreates the
  Activity, so `remember` state inside `setContent` and objects built in `onCreate` are rebuilt.
  An `ActivityResultLauncher` goes with the destroyed instance: the result is delivered to the new
  instance, never to a continuation the old one held. Nothing cancels a `CoroutineScope` you
  created yourself.
- **Fix:** Hold the holder in a lifecycle ViewModel or an application-scoped object, or accept the
  rebuild on purpose: cancel the old scope in `onDispose`, use `rememberSaveable` for user choices,
  reload from cache. Register every `ActivityResultLauncher` in each `onCreate` (before STARTED),
  because the new instance receives the result. In `onDestroy`, resume any pending permission
  waiter with the current permission state.
- **Seen on:** Android Gradle Plugin 8.13, lifecycle-runtime-compose 2.10.0, Compose
  Multiplatform 1.11.1.

## Checklist

- [ ] Drivers and SDK clients built once per process; roots only wire them into the UI
- [ ] Adapters are required parameters with no no-op defaults; key references hoisted
- [ ] expect/actual for facts, interfaces for services, Swift bridges for Swift-only APIs
- [ ] Owner (and generation) checked after every suspension; one lock per shared resource
- [ ] Focused results merged onto live state; presentation flags outside what loads write back
- [ ] Ordered sequences run as one awaited job on a scope that outlives screens
- [ ] One route parser, exact paths, one-slot bus, applied once the UI can navigate
- [ ] Plain holder: recreation chosen on purpose; launchers registered in each `onCreate`
