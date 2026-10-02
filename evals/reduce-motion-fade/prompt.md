---
name: reduce-motion-fade
tags: [kmp-compose-motion]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "With iOS Reduce Motion on, a fallback fade never plays on CMP 1.11 because the OS setting sets the Compose motion duration scale to 0, which cuts every animation."
expected_outcome: "Explains that CMP 1.8+ on iOS sets MotionDurationScale to 0 under Reduce Motion so all Compose animations jump to the end, and runs the one required fade on an Animatable under its own scale of 1f while the flag drops travel and scale."
---
Accessibility pass on our workout app (KMP, CMP 1.11.1, Kotlin 2.4.10; we upgraded from CMP 1.7.3 last month). The design rule: when the user has Reduce Motion on, the workout summary card must not slide or scale in, but it should still fade in over 200 ms, so it does not just pop.

The flag comes from an expect/actual. On iOS it reads `UIAccessibilityIsReduceMotionEnabled()` and listens for the change notification, and we provide it at the root as `LocalReducedMotion`. I logged it: it is true when the switch is on. But with Reduce Motion on, the card just appears. No fade at all. With the switch off, the full slide, scale and fade play fine.

```kotlin
@Composable
fun SummaryCardHost(summary: WorkoutSummary?) {
    val reducedMotion = LocalReducedMotion.current
    val enter = if (reducedMotion) {
        fadeIn(tween(durationMillis = 200))
    } else {
        slideInVertically(spring(dampingRatio = 0.8f)) { it / 3 } +
            scaleIn(spring(dampingRatio = 0.8f), initialScale = 0.94f) + fadeIn()
    }
    AnimatedVisibility(visible = summary != null, enter = enter) {
        summary?.let { SummaryCard(it) }
    }
}
```

What I tried: `tween(600)` and `tween(1500)`, same instant pop. I replaced `AnimatedVisibility` with `animateFloatAsState` driving `Modifier.alpha`, also instant. The card is not visible at first composition (the summary loads async), so it is not the "no enter animation on first composition" case. Why does the fade not play, and how do I get a fade that plays in this mode only?
