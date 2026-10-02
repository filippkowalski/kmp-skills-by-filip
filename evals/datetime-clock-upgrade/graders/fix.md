---
type: llm
weight: 1
focus: last_message
---
Context: CMP 1.11 pulls kotlinx-datetime 0.7.x on iOS while the app declares 0.6.1, so `kotlinx.datetime.Clock.System` stops resolving on iOS. The developer proposes to force 0.6.1 with `strictly`.

PASS if the fix does both:
- Declares kotlinx-datetime 0.7.1 (or a later 0.7.x) explicitly, so every target resolves the same version.
- Changes the imports to `kotlin.time.Clock` and `kotlin.time.Instant`, and keeps `TimeZone`, `LocalDate` and `todayIn` from `kotlinx.datetime`.
It must also advise against forcing 0.6.1, or at least not recommend it.

FAIL if any of these is true:
- It recommends forcing, `strictly` pinning or excluding kotlinx-datetime to stay on 0.6.x (this downgrades a library that CMP 1.11 needs).
- It only changes the version and keeps the `kotlinx.datetime.Clock` / `Instant` imports.
- It only changes the imports and keeps 0.6.1 declared (Android would then mix `kotlin.time.Instant` with 0.6.1 APIs).
- It only suppresses the deprecation, or wraps `Clock` in an expect/actual.
