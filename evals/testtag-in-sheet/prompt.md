---
name: testtag-in-sheet
tags: [kmp-agent-device-testing]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "testTags map to resource-ids on screens but not inside a ModalBottomSheet or AlertDialog, because those draw in their own window outside the root testTagsAsResourceId flag."
expected_outcome: "Explains that sheets, dialogs, menus and popups are separate windows whose semantics tree does not inherit the root flag, and sets testTagsAsResourceId on the first node inside every such window through one shared modifier."
---
We drive our Android shop app from a test script with `adb shell uiautomator dump` and tap by resource-id. Every Compose element has a `testTag`, and we set the flag once at the root:

```kotlin
// MainActivity.onCreate
setContent {
    Box(Modifier.fillMaxSize().semantics { testTagsAsResourceId = true }) {
        ShopApp(graph)
    }
}

// Checkout screen
PrimaryButton(stringResource(Res.string.checkout), Modifier.testTag("checkout_button")) { showPay = true }

if (showPay) {
    ModalBottomSheet(onDismissRequest = { showPay = false }) {
        Column(Modifier.padding(24.dp)) {
            PaymentMethods(methods, Modifier.testTag("payment_methods"))
            PrimaryButton(stringResource(Res.string.pay_now), Modifier.testTag("pay_button"), onClick = onPay)
        }
    }
}
```

On the cart and checkout screens every tag shows up as a resource-id in the dump. With the payment sheet open, the dump shows the sheet's texts ("Pay now", the card names) but `resource-id=""` on all of them. Same inside our "Remove item?" `AlertDialog`. It is not timing: I dump 3 seconds after the sheet opens. It is not the button: the same `PrimaryButton` with a tag works on the checkout screen. CMP 1.11.1, material3 1.9.0, Kotlin 2.4.10, Android 15.

What is different about the sheet, and what is the clean fix for the whole app?
