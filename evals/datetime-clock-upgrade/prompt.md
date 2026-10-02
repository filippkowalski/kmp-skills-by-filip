---
name: datetime-clock-upgrade
tags: [kmp-compose-upgrade]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "After a CMP 1.11 and Kotlin 2.4 bump, Clock.System stops resolving on iOS only; CMP pulls kotlinx-datetime 0.7 on iOS, and 0.7 moved Clock and Instant to kotlin.time."
expected_outcome: "Explains that iOS resolves kotlinx-datetime 0.7.x through CMP 1.11 (material3) while Android keeps 0.6.1, and 0.7 moved Clock and Instant to kotlin.time; declares 0.7.1 and imports kotlin.time.Clock and kotlin.time.Instant."
---
I upgraded our banking app from Kotlin 2.1.21 / CMP 1.7.3 to Kotlin 2.4.10 / CMP 1.11.1 in one branch. Android compiles and all JVM tests pass. iOS does not: `:shared:compileKotlinIosSimulatorArm64` fails with `Unresolved reference 'System'` on every `Clock.System` call (14 places). I did not touch kotlinx-datetime in this branch.

```toml
[versions]
kotlin = "2.4.10"
compose-multiplatform = "1.11.1"
kotlinx-datetime = "0.6.1"
```

```kotlin
import kotlinx.datetime.Clock
import kotlinx.datetime.Instant
import kotlinx.datetime.TimeZone
import kotlinx.datetime.todayIn

fun isStatementOverdue(dueAt: Instant): Boolean = Clock.System.now() > dueAt

fun todayLocal() = Clock.System.todayIn(TimeZone.currentSystemDefault())
```

`commonMain` declares `implementation(libs.kotlinx.datetime)`. I tried a clean build, deleting `~/.konan` and all build dirs, and Invalidate Caches. How can the same common code compile for Android and not for iOS? My first idea is to force kotlinx-datetime to 0.6.1 with `strictly`. Is that the right fix?
