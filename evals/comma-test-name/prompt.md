---
name: comma-test-name
tags: [kmp-testing-strategy]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "A new commonTest file compiles on the JVM but breaks the iOS test compile, because Kotlin/Native rejects a comma in a backticked test name; the error text is not available."
expected_outcome: "Points at the backticked test name that contains a comma (Kotlin/Native rejects commas in declaration names, the JVM accepts them), renames it without the comma, and runs the Native test compile before merge."
---
A PR that only added one test file broke the iOS job of our notes app. `./gradlew :shared:testDebugUnitTest` is green locally and in CI (61 tests, including the 3 new ones). The iOS job fails at `:shared:compileTestKotlinIosSimulatorArm64`, but our CI log limit cut the output, so we only see `Compilation finished with errors` and not the message. I have no Mac this week to reproduce it. `Note` and `merge` live in commonMain and older tests that use them compile on iOS. Here is the new file (Kotlin 2.4.10, kotlinx-coroutines-test 1.10.1):

```kotlin
package app.notes.sync

import kotlinx.coroutines.test.runTest
import kotlin.test.Test
import kotlin.test.assertEquals

class NoteMergeTest {
    private fun note(id: String, body: String, at: Long) = Note(id = id, body = body, updatedAt = at)

    @Test fun `keeps the local edit when the server copy is older`() = runTest {
        val merged = merge(local = note("a", "local", 20), remote = note("a", "remote", 10))
        assertEquals("local", merged.body)
    }

    @Test fun `two offline edits of one note, newest wins`() = runTest {
        val merged = merge(local = note("a", "phone", 30), remote = note("a", "tablet", 40))
        assertEquals("tablet", merged.body)
    }

    @Test fun `deleted on the server beats an older local edit`() = runTest {
        val merged = merge(local = note("a", "x", 5), remote = note("a", "", 9).copy(deleted = true))
        assertEquals(true, merged.deleted)
    }
}
```

What in this file can break only the iOS compile, and how do we stop this kind of thing from slipping through again?
