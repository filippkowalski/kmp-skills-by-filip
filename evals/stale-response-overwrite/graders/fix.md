---
type: llm
weight: 1
focus: last_message
---
Context: `load(accountId)` copies `state` before two suspending calls and assigns that copy back afterwards, with no check that the account is still active. The developer asks whether a `Mutex` around `load()` fixes both bugs.

PASS if the fix does both, and does not endorse a `Mutex` alone as the fix:
1. Ownership: capture the owner before the calls (the `accountId`, plus a generation counter for repeated taps), and after each suspension drop the result if `state.activeAccountId != accountId` or a newer load started. Cancelling the previous load job on each switch is an acceptable addition or alternative for this part.
2. Merge onto live state: write only the fields the load owns onto the current state at write time, for example `state = state.copy(profile = profile, balance = balance, isLoading = false)` read after the suspension, `_state.update { it.withAccount(read) }`, or a focused result type plus a merge function. Never assign from the copy taken before the calls. Presentation flags such as `isExportSheetOpen` stay out of what the load writes.
A `Mutex` used together with an owner check inside the lock is fine.

FAIL if any of these is true:
- It says a `Mutex` around `load()` fixes both bugs, or the fix is only a Mutex.
- It only cancels the previous job and still writes `s.copy(...)`.
- It only debounces taps or disables the chip while loading.
- It uses `Dispatchers.Main.immediate`, `@Volatile` or `runBlocking` as the fix.
- It adds harmful advice, such as catching `CancellationException` or wrapping the suspend calls in plain `runCatching`.
