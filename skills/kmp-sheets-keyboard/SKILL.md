---
name: kmp-sheets-keyboard
description: Material ModalBottomSheet behaviour, window insets, the soft keyboard (IME), text field focus and caret, back handling and main-window overlays in Compose Multiplatform apps on iOS and Android. Use when a sheet jitters or will not settle, a list in a sheet fights the dismiss drag, a sheet runs under the status bar, something opens behind a sheet, a blocking sheet can still be dragged away, a nested scroll crashes, the keyboard covers an input or pans the window, focus or the iOS keyboard does not appear, the caret lands at the start, or back goes to the wrong screen. Triggers on ModalBottomSheet, sheetState, confirmValueChange, ModalBottomSheetProperties, LocalOverscrollFactory, verticalScroll, imePadding, WindowInsets.ime, adjustResize, windowSoftInputMode, FocusRequester, requestFocus, LocalSoftwareKeyboardController, BasicTextField, TextFieldValue, BackHandler, NavigationEventHandler, predictive back, overlay, clearAndSetSemantics.
---

# Sheets, keyboard, focus and back in Compose Multiplatform

General traps in Material 3 sheets, insets and input on both platforms. Most expensive first.
Test tags inside sheet, dialog and popup windows: see kmp-agent-device-testing.

## 1. A ModalBottomSheet whose content is near its maximum height never settles

**Symptom:** the sheet jumps up and down forever. A single screenshot looks correct.
**Cause:** Material sizes the sheet from its content. Content within about a drag handle's height of
the sheet maximum makes it re-solve its anchors every frame. Content that fills the whole allowance
loops the same way when the keyboard opens.
**Fix:** cap fixed-height content at about 85% of the allowance. Forms scroll instead of filling. (A
very short content-sized sheet has the opposite problem: any downward drag dismisses it.)

```kotlin
ModalBottomSheet(onDismissRequest = onClose) {
    Column(Modifier.fillMaxWidth().fillMaxHeight(0.85f)) {
        Header(); Column(Modifier.weight(1f).verticalScroll(rememberScrollState())) { Rows() }
    }
}
```

To check, compare two screenshots taken one second apart. **Seen on:** CMP 1.11.1, material3 1.9.0.

## 2. Scrolling content in a sheet and drag-to-dismiss read the same gesture

**Symptom:** a drag moves the sheet and rubber-bands the list at once. A long list is dismissed
instead of scrolled.
**Cause:** list scroll and the dismiss drag both take downward drags. Compose draws the sheet on iOS
too, so both platforms get the Material drag model. A list with no scroll range sends every drag into
overscroll while the sheet moves.
**Fix:** put long lists on a page. In a sheet whose content usually fits, remove overscroll with
`CompositionLocalProvider(LocalOverscrollFactory provides null) { Content() }`. Overflowing content
still scrolls. **Seen on:** CMP 1.11.1, Android and iOS.

## 3. A sheet is a separate window: insets, z-order and colors change

**Symptom:** a tall sheet runs under the status bar or the Dynamic Island. A full-screen composable
opened from inside the sheet appears behind it. Sheets show a lilac tint that is not in the palette.
**Cause:** `ModalBottomSheet` draws in its own window. Insets read inside it are measured from the
sheet's top edge. Main-window content stays under that window (a second sheet stacks correctly). The
container uses `surfaceContainerLow`, and `lightColorScheme()` defaults the `surfaceContainer*` roles
to baseline purple tones.
**Fix:** read the status bar inset in the caller. Close the sheet before opening a main-window screen.
Set all five `surfaceContainer*` roles, or pass `containerColor`.

```kotlin
val top = WindowInsets.statusBars.asPaddingValues().calculateTopPadding()   // outside the sheet
ModalBottomSheet(onDismissRequest = onClose) {
    BoxWithConstraints {                     // the sheet follows content height
        Box(Modifier.height(maxHeight - top - 10.dp)) { Content(onOpen = { closeSheet(); openScreen() }) }
    }
}
```

**Seen on:** CMP 1.11.1, material3 1.9.0, iOS and Android.

## 4. If a sheet must not close, onDismissRequest alone does not stop it

**Symptom:** a blocking sheet (forced update, required consent) still slides away on a drag, or back
closes it or reaches the screen behind.
**Cause:** `onDismissRequest` is only a callback; the drag to `Hidden` still runs. Back goes to the
sheet window and follows `shouldDismissOnBackPress`.
**Fix:**
```kotlin
// inside a @Composable with @OptIn(ExperimentalMaterial3Api::class, ExperimentalComposeUiApi::class)
val state = rememberModalBottomSheetState(skipPartiallyExpanded = true,
    confirmValueChange = { !(required && it == SheetValue.Hidden) })
BackHandler(enabled = required) {}         // rule 9; outside the sheet content
ModalBottomSheet(onDismissRequest = { if (!required) onLater() }, sheetState = state,
    dragHandle = if (required) null else ({ BottomSheetDefaults.DragHandle() }),
    properties = ModalBottomSheetProperties(shouldDismissOnBackPress = !required)) { Content() }
```

**Seen on:** CMP 1.11.1, material3 1.9.0.

## 5. A vertical scroller inside a vertical scroller crashes

**Symptom:** a screen crashes on open with an infinite-height constraint error.
**Cause:** a `verticalScroll` column or `LazyColumn` inside a vertically scrolling parent is measured
with infinite height.
**Fix:** one scroller per axis. A child of a scrolling parent does not scroll itself. A `LazyColumn`
in a sheet or scrolling column gets a bounded height. **Seen on:** CMP 1.11.1.

## 6. Keyboard: declare adjustResize on Android, then pick one IME layout per screen

**Symptom:** the keyboard covers the input, pushes the header off screen, or leaves a double gap.
**Cause:** an edge-to-edge window does not shrink for the keyboard (Android with `adjustResize`, and
iOS): the IME is an inset each screen must consume. A global `adjustResize` uncovers every screen.
**Fix:** declare `adjustResize` on Android, then pick ONE layout per screen. A field pinned to the
bottom: pad the root column by nav bars and IME together, with the field a direct child (an overlay
`Box` escapes the padding). A scrolling form: `imePadding()` on the scroll container.

```kotlin
Column(Modifier.fillMaxHeight().statusBarsPadding()
    .windowInsetsPadding(WindowInsets.navigationBars.union(WindowInsets.ime))) {
    Content(Modifier.weight(1f)); SearchField()          // pinned field, direct child
}
LazyColumn(Modifier.fillMaxSize().imePadding()) { formFields() }   // scrolling form
```

The same modifiers work on iOS, where `WindowInsets.ime` carries the keyboard height. Double padding
comes from manual `WindowInsets.ime.getBottom()` math or a second consumer. CMP 1.11.1 also has
`Modifier.fitInside(WindowInsetsRulers.Ime.current)` in common code; not tested on iOS in this pack.
**Seen on:** CMP 1.7.3 and 1.11.1, Android 15+ edge-to-edge.
See also: edge-to-edge in android/skills (Android IME setup, `fitInside` with insets rulers, Scaffold insets).

## 7. Focus requests: guard them, retry on iOS, show the keyboard explicitly

**Symptom:** on iOS a caret blinks but no keyboard appears, or the app crashes at screen entry.
**Cause:** `requestFocus()` throws when the requester is not attached yet. iOS can drop a request made
while the field is still animating in. Compose focus and the UIKit keyboard are separate: a field can
hold focus while the keyboard never rose. (A throw from an effect ends the iOS process: see
kmp-ios-build.)
**Fix:**

```kotlin
val requester = remember { FocusRequester() }; var focused by remember { mutableStateOf(false) }
val keyboard = LocalSoftwareKeyboardController.current
LaunchedEffect(Unit) {
    repeat(4) { if (!focused) runCatching { requester.requestFocus() }; delay(200) }  // starting point: tune to your entrance
    if (focused) keyboard?.show()
}
DisposableEffect(Unit) { onDispose { focusManager.clearFocus() } }   // hide on leave
BasicTextField(value, onValueChange,
    Modifier.focusRequester(requester).onFocusChanged { focused = it.isFocused })
```

- `LaunchedEffect(Unit)` runs again whenever the field re-enters composition. To focus only on a user
  action, key the effect on a counter that the action increments.
- A text field that stays composed under an overlay keeps focus and the keyboard. When the overlay
  opens, call `focusManager.clearFocus(force = true)` and `keyboard?.hide()`.

**Seen on:** CMP 1.7.3 and 1.11.1, iOS 26 simulators.

## 8. BasicTextField(String) starts the caret at index 0

**Symptom:** the user returns to a prefilled field, types, and the text goes in front of the old text.
**Cause:** the `String` overload keeps its own selection and seeds it at 0.
**Fix:** hold a `TextFieldValue` with the caret at the end; re-seed only on outside changes.

```kotlin
var field by remember { mutableStateOf(TextFieldValue(text, TextRange(text.length))) }
LaunchedEffect(text) { if (text != field.text) field = TextFieldValue(text, TextRange(text.length)) }
BasicTextField(field, { field = it; onText(it.text) })
```

**Seen on:** CMP 1.11.1.

## 9. Back: the most recent enabled handler wins, and iOS has no back button

**Symptom:** back switches tabs behind an open overlay, or an outer handler never runs.
**Cause:** back handlers stack on a dispatcher, and the last enabled one gets the event. iOS has no
system back button.
**Fix:** use the common `androidx.compose.ui.backhandler.BackHandler` (`ui-backhandler`, needs
`@OptIn(ExperimentalComposeUiApi::class)`) instead of a hand-written expect/actual:
`BackHandler(enabled = overlayUp) { closeOverlay() }`. In CMP 1.11.1 both `BackHandler` and
`PredictiveBackHandler` there are deprecated (a warning) in favour of `NavigationEventHandler`
(androidx.navigationevent). `BackHandler` still works and is the smallest common call; use
`NavigationEventHandler` if you already depend on navigationevent, and plan the move either way.
Compose an overlay's handler after the screen's handler, enabled while the overlay is visible.
Enable nested handlers only while they have something to pop. Give iOS pages a visible back
control. With targetSdk 36, predictive back needs no extra code.
**Seen on:** CMP 1.11.1 (`ui-backhandler` metadata), activity-compose 1.12.2, targetSdk 36, Android 17.
See also: navigation-event in android/skills (`NavigationBackHandler`, dialog and sheet dispatchers, the move off `BackHandler`).

## 10. If overlays are Box siblings (no Dialog, no nav library), they are not dialogs

**Symptom:** a full-screen overlay draws under the content, taps pass through it, or a screen reader
reads the content below.
**Cause:** siblings in a `Box` stack in declaration order. Only dialog windows block input and
accessibility underneath.
**Fix:** emit the overlay last, give it back handling (rule 9), swallow taps, clear semantics below.

```kotlin
Box(Modifier.fillMaxSize()) {
    Screen(if (overlayUp) Modifier.clearAndSetSemantics {} else Modifier)
    if (overlayUp) Box(Modifier.fillMaxSize().clickable(
        interactionSource = remember { MutableInteractionSource() }, indication = null) {}) { Overlay() }
}
```

`AnimatedVisibility` keeps composing content during exit. Keep the last non-null model so the page
does not go blank while it animates out. **Seen on:** CMP 1.7.3 and 1.11.1.

## Checklist

- [ ] Sheet content stays about 15% below the maximum; forms scroll; long lists are pages; sheets that
      fit use `LocalOverscrollFactory provides null`.
- [ ] Status bar inset read outside the sheet; sheet closed before a main-window screen opens; all
      `surfaceContainer*` roles set.
- [ ] Blocking sheets (if any): `confirmValueChange`, `shouldDismissOnBackPress = false`, no handle,
      outer back handler.
- [ ] One vertical scroller per axis; lazy lists in sheets have a bounded height.
- [ ] `adjustResize` declared; each input screen uses root IME padding OR `imePadding()` on its form.
- [ ] `requestFocus()` guarded, retried on iOS, followed by `keyboard?.show()`; prefilled fields use
      `TextFieldValue` with the caret at the end.
- [ ] Common `BackHandler` (or `NavigationEventHandler`) on every overlay; iOS pages have a back control.
- [ ] Box-sibling overlays (if any) are emitted last, swallow taps and clear semantics below.
