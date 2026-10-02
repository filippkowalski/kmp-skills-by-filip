---
type: llm
weight: 1
focus: last_message
---
Context: testTags inside a `ModalBottomSheet` and an `AlertDialog` do not become Android resource-ids, because those draw in separate windows that the root `testTagsAsResourceId` flag does not reach. The developer asks for a clean fix for the whole app.

PASS if the fix applies `semantics { testTagsAsResourceId = true }` again inside every sheet, dialog, dropdown menu and popup window, on the first node inside the window (for example the sheet content's root `Column`, `AlertDialog(modifier = ...)`, `DropdownMenu(modifier = ...)`) or on the `ModalBottomSheet(modifier = ...)`. The whole app must be covered: through one shared modifier (for example an expect/actual that is a no-op on iOS), wrapper composables, or a stated plan to find every `ModalBottomSheet`, `Dialog`, `DropdownMenu` and `Popup(`.

FAIL if any of these is true:
- It fixes only this one sheet, with no plan for the dialog and the other windows.
- It switches the script to text or content-description selectors, or adds `contentDescription` only for tests (this changes what screen readers announce).
- It replaces the sheet or dialog with an in-window overlay only to make tests work.
- It only suggests moving to Espresso or Compose UI tests.
