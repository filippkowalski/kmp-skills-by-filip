---
name: kmp-testing-strategy
description: Write and run shared tests in a Kotlin Multiplatform + Compose Multiplatform app so they compile and run on every target. Use when writing commonTest tests, when a KMP test task passes on one target and fails or builds nothing on the other, when a test touches Compose resources, expect/actual classes or coroutines, or before trusting a green test run. Triggers - commonTest, iosSimulatorArm64Test, testDebugUnitTest, allTests, backticked test names, Kotlin/Native test compile, expect class, getString in tests, runTest, withTimeout, virtual time, Ktor MockEngine.
---

# KMP testing strategy

`commonTest` compiles once per target: for the JVM (`testDebugUnitTest`) and for Kotlin/Native
(`iosSimulatorArm64Test`). Each compiler accepts things the other rejects. When a target's test
source set fails to compile, that target runs zero tests, and the other target stays green.

## 1. One target's green run says nothing about the other

**Symptom:** tests pass on one target for weeks while the other target's test source set does not
compile, or compiles and builds no tests.
**Cause:** each target compiles its own source sets and resolves its own dependency graph. A
transitive dependency can resolve to different versions per target, so the same common code can
compile on Android and fail on iOS.
**Fix:** run both test tasks as separate commands and compare the counts. The same `commonTest`
must report the same non-zero count on both. Compile iOS whenever an `iosMain` file, an `expect`
or a dependency version changes.

```sh
./gradlew :app:testDebugUnitTest
./gradlew :app:iosSimulatorArm64Test
./gradlew :app:compileKotlinIosSimulatorArm64     # after iosMain, expect or version changes
```

See kmp-compose-upgrade for the kotlinx-datetime 0.7 case (`Clock.System` resolved on Android only).
**Seen on:** Kotlin 2.1.21 and 2.4.10, CMP 1.7.3 and 1.11.1.

## 2. Kotlin/Native rejects a comma in a backticked test name

**Symptom:** `iosSimulatorArm64Test` fails to compile; `testDebugUnitTest` is green.
**Cause:** the JVM accepts `` fun `a, b`() ``; Kotlin/Native rejects `,` in a declaration name
with `error: name contains illegal characters: ",".` New commas return with every new test file
unless the Native test compile runs.
**Fix:** no commas in test names. Write "and" or "but".

```kotlin
@Test fun `adding an item and syncing twice stores it once`() { /* ... */ }
```

**Seen on:** Kotlin 2.1.21 and 2.4.10 (error text from `kotlinc-native` 2.4.10).

## 3. An `expect class` with platform-only constructors breaks commonTest

**Symptom:** `testDebugUnitTest` builds no tests at all; iOS tests pass.
**Cause:** common code can rely only on the constructors the `expect` declares, and an `expect`
cannot declare a `Context` parameter. The Android `actual` takes a `Context`, the iOS one takes
nothing, so a common call such as `SecureStore()` compiles only where the `actual` happens to match.
**Fix:** when actuals disagree on construction, use an interface, one class per platform, and a
fake in tests. Build the platform classes in the platform entry points.

```kotlin
interface SecureStore {
    fun get(key: String): String?
    fun put(key: String, value: String)
}
class AndroidSecureStore(context: Context) : SecureStore { /* ... */ }   // androidMain
class IosSecureStore : SecureStore { /* ... */ }                          // iosMain, Keychain
private class InMemorySecureStore : SecureStore {                         // commonTest
    private val map = mutableMapOf<String, String>()
    override fun get(key: String) = map[key]
    override fun put(key: String, value: String) { map[key] = value }
}
```

**Seen on:** Kotlin 2.4.10.

## 4. If shared logic reads Compose resources, an unconfigured Android JVM test cannot resolve them

**Symptom:** a test passes on `iosSimulatorArm64Test` and always fails on `testDebugUnitTest`.
**Cause:** the code under test reaches `getString(Res.string.x)`. On Android, the resource reader
opens the file from the app's assets through a `Context`, then falls back to a class-loader
stream. A plain Android JVM unit test sets up no `Context` or assets, so the lookup fails with
`MissingResourceException` unless the resource files are on the test classpath.
**Fix:** keep resource reads out of the logic under test (pass the resolved string in, or read it
in the UI layer). If an assertion only proves the platform limit, drop it. The generated maps
such as `Res.allStringResources` are plain Kotlin and work on both targets.
See kmp-strings-localization for the `getString` threading rule.
**Seen on:** CMP 1.11.1 (`DefaultAndroidResourceReader` in components-resources-android).

## 5. If a test waits on real dispatchers, `withTimeout` inside `runTest` runs on virtual time

**Symptom:** a wait-until-true helper times out before code on `Dispatchers.Default` finishes.
**Cause:** inside `runTest`, `delay` advances virtual time instantly, so a 5 s `withTimeout`
passes in microseconds of real time while real threads are still working.
**Fix:** run the polling loop on a real dispatcher.

```kotlin
private suspend fun waitUntil(condition: () -> Boolean) = withContext(Dispatchers.Default) {
    withTimeout(5_000) { while (!condition()) delay(10) }
}
```

**Seen on:** kotlinx-coroutines-test 1.10.1.

## 6. If you use Ktor, test network code through the real client on `MockEngine`

A pattern, not a trap. Build the real API client with `HttpClient(MockEngine { ... })` and feed it
response bodies, including values this build has never seen. Assert what the caller receives, not
what a helper does. See kmp-api-contracts for unknown enum values in responses.

```kotlin
val engine = MockEngine { req ->
    respond(bodies[req.url.encodedPath] ?: "{}",
        if (req.url.encodedPath in bodies) HttpStatusCode.OK else HttpStatusCode.NotFound,
        headersOf(HttpHeaders.ContentType, ContentType.Application.Json.toString()))
}
val api = ApiClient(httpClient = HttpClient(engine) { install(ContentNegotiation) { json() } })
```

A regression test proves the fix only if it fails when the bug is put back. Run it once against
the old code.

## Related

- See kmp-ios-build: the iOS test executable links without what Xcode adds for the app. If it
  does not link, zero iOS tests ran.

## Checklist

- [ ] `testDebugUnitTest` and `iosSimulatorArm64Test` both run; counts equal and non-zero
- [ ] iOS compiled after any `iosMain`, `expect` or dependency version change
- [ ] No commas in backticked test names
- [ ] No `expect class` whose actuals need different constructors
- [ ] No `getString` reached from JVM-tested logic
- [ ] Polling helpers leave virtual time (`withContext(Dispatchers.Default)`)
- [ ] Network tests go through the real client on `MockEngine`
- [ ] Each regression test failed once against the old code
