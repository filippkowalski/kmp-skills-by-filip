---
name: stale-response-overwrite
tags: [kmp-app-architecture]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "A load copies state before suspending and writes the copy back without an ownership check; a fast account switch shows the previous account and an open sheet closes itself."
expected_outcome: "Explains the missing ownership check after suspension and the stale snapshot that s.copy() writes back, and fixes it with an owner or generation check plus a merge of only the loaded fields onto live state; a Mutex alone does not fix it."
---
In our banking app (KMP, CMP 1.11.1, Kotlin 2.4.10, kotlinx-coroutines 1.10.1; no ViewModels, a plain state holder whose scope runs on `Dispatchers.Main`) users switch between a personal and a business account with a chip in the header. QA found two bugs that I think are related:

1. Tap Business, then quickly Personal, then Business again. Sometimes the app ends up on Personal (chip, name and balance), although Business was tapped last.
2. If the user opens the "Export statement" sheet while an account is still loading, the sheet sometimes closes by itself when the load finishes.

```kotlin
class AccountHolder(private val api: BankApi, private val scope: CoroutineScope) {
    var state by mutableStateOf(AccountState())
        private set

    fun switchTo(accountId: String) {
        state = state.copy(activeAccountId = accountId, isLoading = true)
        scope.launch { load(accountId) }
    }

    fun openExport() { state = state.copy(isExportSheetOpen = true) }

    private suspend fun load(accountId: String) {
        val s = state
        val profile = api.profile(accountId)    // 200 ms to 3 s
        val balance = api.balance(accountId)
        state = s.copy(profile = profile, balance = balance, isLoading = false)
    }
}
```

Everything runs on the main thread, so I do not see how there can be a race. A teammate suggests a `Mutex` around `load()`. Would that fix both bugs? If not, what is the right fix?
