---
name: getstring-deadlock
tags: [kmp-strings-localization]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "Cold-start ANR: a Main-dispatched state holder calls suspend getString while composition reads the same string with stringResource, which blocks the main thread."
expected_outcome: "Explains that getString on Dispatchers.Main queues the cached resource load on the main thread while stringResource waits for that same load in runBlocking on the main thread, and moves every suspend getString off Main (withContext(Dispatchers.Default) helper)."
---
Our notes app (KMP, CMP 1.11.1, Kotlin 2.4.10, kotlinx-coroutines 1.10.1) freezes on some Android cold starts. The home screen stays on the skeleton list, nothing responds, and after about 5 seconds Android shows "App isn't responding". It happens on maybe 1 in 5 cold starts on a Pixel 6a, never on a warm start, and we have not seen it on iOS.

The main thread in the ANR trace:

```
"main" prio=5 tid=1 Waiting
  at jdk.internal.misc.Unsafe.park(Native Method)
  at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt)
  at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt)
  ...
  at org.jetbrains.compose.resources.StringResourcesKt.stringResource(StringResources.kt)
  at app.notes.home.HomeTopBarKt.HomeTopBar(HomeTopBar.kt:21)
```

The code:

```kotlin
class HomeHolder(private val repo: NotesRepository) {
    private val scope = MainScope()
    var state by mutableStateOf<HomeState>(HomeState.Loading)
        private set

    fun start() = scope.launch {
        val notes = repo.recentNotes()            // Room, switches to Dispatchers.IO inside
        val title = getString(Res.string.home_title, notes.size)
        state = HomeState.Ready(title, notes)
    }
}
// MainActivity.onCreate: holder.start(); setContent { NotesApp(holder) }

@Composable
fun HomeTopBar(state: HomeState) {
    val title = (state as? HomeState.Ready)?.title ?: stringResource(Res.string.home_title, 0)
    TopAppBar(title = { Text(title) })
}
```

We checked that the Room query really runs on IO, and the strings file is small. Why would `stringResource` block the main thread forever, and what is the proper fix? I would rather not move the whole holder off the main thread.
