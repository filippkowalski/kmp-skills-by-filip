---
type: llm
weight: 2
focus: last_message
---
Context: on iOS with Reduce Motion on, a CMP 1.11.1 app picks `fadeIn(tween(200))` instead of slide and scale. The fade never plays; the card appears at once. Longer tweens and an `animateFloatAsState` alpha were also instant. The reduced-motion flag reads true.

PASS if the answer states that the OS setting itself cuts Compose animations:
- Compose runs animations under a `MotionDurationScale` taken from the coroutine or recomposer context.
- CMP (1.8 and later) on iOS sets that scale to 0 while Reduce Motion is on (Android does the same for "Remove animations").
- At scale 0, `AnimatedVisibility`, `animate*AsState`, `Animatable` and transitions jump to their end value on the first frame, whatever the duration. So the fade is cut by the framework, not by this code.

FAIL if any of these is true:
- It blames `AnimatedVisibility` not animating on first composition, or the `visible` logic.
- It says the tween is too short, or the flag is wrong or not reactive.
- It blames recomposition, a missing `key`, or the `CompositionLocal`.
- It says `fadeIn` or alpha animation is broken in CMP 1.11 or on iOS (Skia, Metal, frame clock) without the duration-scale mechanism.
- It says only that "iOS disables animations under Reduce Motion" without naming the Compose motion duration scale being set to 0.
