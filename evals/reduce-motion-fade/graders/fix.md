---
type: llm
weight: 1
focus: last_message
---
Context: under iOS Reduce Motion, CMP 1.11 runs every Compose animation at a motion duration scale of 0, so a 200 ms fallback fade is cut. The design still wants that one fade to play, without slide or scale.

PASS if the fix keeps the reduced-motion flag to drop travel and scale, and plays the one required fade outside the system scale, by one of these:
- An `Animatable` for alpha, animated in a `LaunchedEffect` inside `withContext(<a MotionDurationScale whose scaleFactor is 1f>) { alpha.animateTo(1f, tween(200)) }`, with the alpha applied in `graphicsLayer` or `Modifier.alpha`.
- An equally correct alternative: drive the alpha by hand from frame times (a `withFrameNanos` loop that computes progress), which does not read the motion duration scale.

FAIL if any of these is true:
- It changes durations, or tweaks `AnimatedVisibility`, `MutableTransitionState` or `animate*AsState` (they take no context and are still cut).
- It provides a scale of 1 for the whole app or root composition, so all motion plays again even with Reduce Motion on.
- It tells users to turn Reduce Motion off, or downgrades CMP.
- It adds harmful advice, such as catching `CancellationException` around `animateTo`.
