---
type: llm
weight: 2
focus: last_message
---
Context: a Material 3 `ModalBottomSheet` (CMP 1.11.1, material3 1.9.0) with a tall content-sized `Column` of filter rows bounces by a few dp every frame on some phones and is still on both taller and shorter screens. Removing one group stops it. There are no recompositions in the content and the state does not change.

PASS if the answer states that this is not a state or recomposition loop in the content. Material sizes the sheet from its content, and when the content height is close to the sheet's maximum height (within about a drag handle's height of it), the sheet re-solves its anchors every frame and never settles. That is why only screens where this content nearly fills the sheet bounce, and why removing a group stops it.

FAIL if any of these is true:
- It blames a recomposition loop in `FilterGroup`, `PriceSlider` or `filters`, or unstable lambdas.
- It blames `skipPartiallyExpanded`, a sheet state that is not remembered, or the animation spec.
- It blames the keyboard or IME (there is no text field).
- It blames window insets or double padding, without the near-maximum-height mechanism.
- It says only "a CMP or material3 bug, upgrade", or it lists content height only as one of several unranked guesses.
