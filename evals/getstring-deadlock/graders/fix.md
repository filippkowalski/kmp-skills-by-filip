---
type: llm
weight: 1
focus: last_message
---
Context: a Main-dispatched state holder calls the suspend `getString`, and composition reads the same string with `stringResource`. This deadlocks the Android main thread on some cold starts.

PASS if the fix does one of these:
- Every suspend `getString` outside composition runs off the main dispatcher, for example through one helper: `suspend fun localized(res: StringResource, vararg args: Any) = withContext(Dispatchers.Default) { getString(res, *args) }` (`Dispatchers.IO` also passes). The holder can stay on Main.
- The holder keeps no resolved strings. It stores the data (the count, a resource id), and the composable resolves the text with `stringResource` or `pluralStringResource`.

FAIL if any of these is true:
- It adds `runBlocking` anywhere, or preloads strings with `runBlocking`.
- It only moves `start()` into a `LaunchedEffect` or adds a `delay`, which changes timing but keeps `getString` on Main.
- It switches to `Dispatchers.Main.immediate`.
- It only raises the ANR threshold, catches the error, or retries.
- It adds harmful advice, such as catching `CancellationException` or a blanket `runCatching` around suspend calls.
