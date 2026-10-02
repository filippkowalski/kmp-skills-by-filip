---
type: llm
weight: 2
focus: last_message
---
Context: after bumping to Kotlin 2.4.10 and CMP 1.11.1, with kotlinx-datetime still declared at 0.6.1, `Clock.System` (imported from `kotlinx.datetime.Clock`) fails with "Unresolved reference 'System'" on iOS only. Android compiles. Caches were cleared.

PASS if the answer states both:
- The platforms resolve different kotlinx-datetime majors. The CMP 1.11 upgrade brings kotlinx-datetime 0.7.x in transitively on the iOS side (for example through `compose.material3`, for its date pickers), and Gradle picks 0.7.x there, while Android keeps the declared 0.6.1. Naming "a CMP 1.11 dependency" instead of material3 is fine.
- kotlinx-datetime 0.7 moved `Clock` and `Instant` to `kotlin.time` in the standard library. `kotlinx.datetime.Clock` is left only as a deprecated typealias, which does not expose the nested `System`, so `Clock.System` does not resolve on iOS.

FAIL if any of these is true:
- It blames stale caches, `~/.konan`, or a missing iOS dependency.
- It calls it a Kotlin 2.4 or Kotlin/Native bug.
- It says kotlinx-datetime 0.6.1 does not support Kotlin 2.4 or iOS, without the transitive version split.
- It says only that `Clock.System` is deprecated, without the move to `kotlin.time`.
- It never explains why Android and iOS differ.
