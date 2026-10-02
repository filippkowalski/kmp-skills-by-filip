---
type: llm
weight: 1
focus: last_message
---
Context: a `ModalBottomSheet` bounces forever because its content-sized column ends up near the sheet's maximum height on some screens.

PASS if the fix keeps the content clearly below the sheet's maximum on every screen, by one of these:
- Cap the content at about 85% of the available height (for example `Modifier.fillMaxHeight(0.85f)`, or `heightIn(max = ...)` computed from the available height), with the filter groups in a scrolling region (`Modifier.weight(1f).verticalScroll(...)`) and the Apply button outside that scroll.
- Move this long form out of the sheet onto a full page.

FAIL if any of these is true:
- The content fills exactly the whole sheet height (`fillMaxHeight()` or `fillMaxSize()` at 100%).
- It sets a fixed dp height that is not derived from the available height.
- It only changes `skipPartiallyExpanded`, the sheet state, the animation spec, or `remember` keys.
- It hides filter groups on some devices, or disables dragging.
