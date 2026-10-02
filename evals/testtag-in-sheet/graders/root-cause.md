---
type: llm
weight: 2
focus: last_message
---
Context: an Android Compose app sets `semantics { testTagsAsResourceId = true }` on a `Box` at the root of `setContent`. `uiautomator dump` shows testTags as resource-ids on normal screens, but not for elements inside a `ModalBottomSheet` or an `AlertDialog`, although their texts appear in the dump. Timing and the button component are ruled out.

PASS if the answer states that `ModalBottomSheet`, `AlertDialog`/`Dialog` (and also `DropdownMenu` and `Popup`) draw in their own window, with their own semantics tree root. Their content is not a descendant of the root `Box`, so the `testTagsAsResourceId` flag set there does not reach the nodes inside them.

FAIL if any of these is true:
- It blames merged semantics (the button merging its children) or `useUnmergedTree`.
- It blames timing, the open animation, or uiautomator not reading Compose.
- It says a `contentDescription` is missing.
- It blames R8/ProGuard, build type, or the Android version.
- It says only "the sheet has a different semantics setup" without the separate-window reason.
