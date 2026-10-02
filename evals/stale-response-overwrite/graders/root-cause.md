---
type: llm
weight: 2
focus: last_message
---
Context: a Main-thread state holder runs `load(accountId)`: `val s = state`, then two suspending API calls, then `state = s.copy(profile = ..., balance = ..., isLoading = false)`. Fast account switching sometimes ends on the wrong account, and an export sheet opened during a load closes itself when the load finishes. The developer thinks a single thread rules out a race.

PASS if the answer explains both mechanisms:
- Ownership: while `load` is suspended in `api.profile` / `api.balance`, other coroutines and taps run, even on the main thread, and responses can finish in any order. Nothing checks after the calls that `accountId` is still the active account, so a slower load for an earlier tap writes last.
- Stale snapshot: `val s = state` is taken before the suspension, and `state = s.copy(...)` writes every field of that old snapshot back. This reverts what changed meanwhile: `activeAccountId` (bug 1, the chip jumps back) and `isExportSheetOpen` (bug 2, the sheet closes).

FAIL if any of these is true:
- It calls it a thread-safety problem and proposes `@Volatile`, atomics, `StateFlow` or a lock as the explanation.
- It explains only one of the two mechanisms (for example only the out-of-order responses, and never why the sheet closes).
- It says only "cancel the previous job" without explaining the snapshot overwrite.
- It blames Compose recomposition or `mutableStateOf`.
- It stays at a vague "race condition" without saying what is overwritten and why.
