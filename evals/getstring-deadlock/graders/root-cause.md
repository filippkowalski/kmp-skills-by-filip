---
type: llm
weight: 2
focus: last_message
---
Context: a state holder on `MainScope()` calls the suspend `getString(Res.string.home_title, n)` after a Room query. The top bar composable reads the same string with `stringResource(Res.string.home_title, 0)` while loading. On some Android cold starts the main thread is parked in `runBlocking` inside `stringResource` until an ANR.

PASS if the answer explains the deadlock with both halves:
- `getString` called from a coroutine on `Dispatchers.Main` starts the load for that string in compose-resources' shared cache (a Mutex-guarded async cache) with `async` on the caller's dispatcher, so the loader is queued on the main thread.
- Before that loader runs, composition calls `stringResource` for the same string. `stringResource` uses `runBlocking` on the main thread and waits for the same in-flight load. The load needs the main thread, which is parked, so neither can finish.
The answer may describe the cache in its own words ("the cached deferred for that resource"), but it must say that the load started by `getString` is queued on Main and that `stringResource`'s `runBlocking` waits on it.

FAIL if any of these is true:
- It blames Room, disk or network work on the main thread, or a slow resource file read.
- It says `stringResource` is just slow on cold start, or blames a recomposition loop.
- It blames `MainScope`, `LaunchedEffect` or lifecycle leaks without the shared-load mechanism.
- It says "do not call getString outside composition" or "it is a CMP bug, upgrade" without explaining why it blocks.
- It suggests `Dispatchers.Main.immediate` as the explanation or cure.
