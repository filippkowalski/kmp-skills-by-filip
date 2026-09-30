---
name: kmp-strings-localization
description: Compose Multiplatform resources and localization traps (composeResources, Res.string, getString, stringResource, pluralStringResource, values-xx folders). Use when adding or translating strings, adding a locale, localizing push notifications, iOS permission prompts or launcher shortcuts, formatting dates, numbers or currency, or when text shows a backslash, a raw key, English in a translated build, a wrong plural, or the app hangs on a string read.
---

# KMP strings and localization

Library and platform behaviour that breaks strings in Compose Multiplatform apps on Android and iOS. Most of these fail silently: the build is green and one language renders wrong.

## 1. `getString` on the main dispatcher can deadlock with `stringResource`

**Symptom:** cold start stays on the first frame or on skeletons until Android shows an ANR. The main thread is parked in `stringResource`, inside `runBlocking`.

**Cause:** compose-resources caches each load behind a `Mutex` and starts the loader with `async` on the caller's dispatcher. A coroutine on `Dispatchers.Main` that calls the suspend `getString` queues the loader on the main thread. Composition then reads the same resource with `stringResource`, which uses `runBlocking` and parks the main thread. The loader waits for a main-thread slot that never comes.

**Fix:** send every suspend string read outside composition through one helper that leaves Main.

```kotlin
suspend fun localized(res: StringResource, vararg args: Any): String =
    withContext(Dispatchers.Default) { getString(res, *args) }
```

**Seen on:** CMP 1.7.3 on Android, kotlinx-coroutines 1.10.1. The same `AsyncCache` and the blocking `ResourceState` are in CMP 1.11.1, on Android and in the iOS artifact.

## 2. If you send loc-key pushes or ship launcher shortcuts: Android cannot read composeResources

Applies only if you send FCM messages with `title_loc_key` / `body_loc_key`, or ship `shortcuts.xml`.

**Symptom:** a push notification shows the raw key or English text. Launcher shortcut labels stay English in a translated app.

**Cause:** CMP packages resources as assets, not as Android `res/`. FCM resolves `title_loc_key` / `body_loc_key` through the package resource table while the app is not running. `shortcuts.xml` labels also resolve only from `res/`.

**Fix:** mirror those keys into `androidMain/res/values*/strings.xml` for every locale. Keep the Compose copy as the source and compare the two copies in a check.
- Keep push keys free of placeholders. loc-args are positional and `body_loc_key` cannot select a `<plurals>`, so a counted body is wrong in languages with `few` / `many`.
- The escaping rules differ (rule 4). Android `res/` needs `\'` for a straight apostrophe. A curly `’` is valid in both files.
- No Kotlin code references these keys, so a rename breaks no build. Pin the names with the generated map, which loads no resources:

```kotlin
@OptIn(ExperimentalResourceApi::class)
@Test fun pushKeysAreFlatStrings() {
    val keys = listOf("push_reminder_title", "push_reminder_body")
    assertEquals(emptyList(), keys.filterNot { it in Res.allStringResources })
    assertEquals(emptyList(), keys.filter { it in Res.allPluralStringResources })
}
```

**Seen on:** CMP 1.7.3 and 1.11.1, firebase-bom 33.7.0.

## 3. iOS system code cannot read composeResources

**Symptom:** permission prompts stay English. Quick actions show raw lookup tokens. A push shows the `loc-key` text on the lock screen.

**Cause:** APNs resolves `loc-key` against the main bundle's `Localizable.strings`. Info.plist keys (`NSCameraUsageDescription`, `NSFaceIDUsageDescription`, `UIApplicationShortcutItemTitle`, ...) resolve against `InfoPlist.strings`. An `.lproj` file that is not in the app target's Resources phase is silently missing from the app, even when the Xcode project references it.

**Fix:**
- Add `xx.lproj/Localizable.strings` and `xx.lproj/InfoPlist.strings` per locale, and make sure they are in the Resources phase and tracked in version control. See kmp-ios-build: project.yml/XcodeGen.
- Keep readable English as the Info.plist value of top-level keys. It is the fallback. A lookup token as the plist value renders raw when the lookup fails.
- Generate the `.strings` values from the Compose catalog. Decode XML entities so characters such as U+202F survive.
- Check the built app, not the source tree: `plutil -p <App>.app/de.lproj/Localizable.strings`.

**Seen on:** Xcode 16, iOS 18 target, CMP 1.7.3 and 1.11.1.

## 4. Android-style escapes render literally

**Symptom:** the screen shows `Let\'s go` with the backslash.

**Cause:** the Compose Gradle plugin unescapes only `\n`, `\t`, `\uXXXX` and `\\`. `\'` and `\"` stay as text. Android `res/` rules do not apply.

**Fix:** write `’` and the locale's quotation marks directly. Re-check escaping when you copy strings between `composeResources` and Android `res/`.

**Seen on:** CMP 1.7.3 (rendered), CMP 1.11.1 (same unescape set in the plugin).

## 5. Format arguments are a regex, not printf

**Symptom:** `%%` shows two percent signs. `%s` or `%.1f` shows literally.

**Cause:** `stringResource` and `getString` replace only `%(\d+)\$[ds]` (CMP 1.11.1) or `%(\d)\$[ds]` (CMP 1.7.3, one digit, so more than 9 arguments break). Nothing else is interpreted. A bare `%` renders as `%`, so `%1$d%` gives `40%`.

**Fix:**
- Use only `%1$s` / `%1$d`. Do not escape `%` in composeResources.
- Format numbers with the platform formatter (rule 9) and pass a string.
- Android's own `Context.getString(id, args)` uses Java formatting, where a bare `%` throws. A string that moves into `res/` needs `%%` there.
- Check the regex again after a CMP upgrade.

**Seen on:** CMP 1.7.3 and 1.11.1.

## 6. If you ship region variants: region folders do not fall back to other regions

Applies if you translate into a regional variant (`pt-BR`, `es-MX`, `zh-Hans`).

**Symptom:** a `pt-PT` device shows the default language although `values-pt-rBR` exists.

**Cause:** CMP picks, in this order: language + exact region, then the language folder with no region, then the unqualified folder. There is no "any region of this language" step. The region qualifier must be `rXX`. BCP-47 script folders such as `values-b+zh+Hans` failed the build on CMP 1.7.3.

**Fix:** use language-only folders (`values-pt`) and put the variant in the text. Add a region folder only next to a language-only folder. The iOS `.lproj` name can still carry the region (`pt-BR.lproj`).

**Seen on:** CMP 1.7.3; same `filterByLocale` order in CMP 1.11.1.

## 7. Plurals need the language's CLDR categories

**Symptom:** counts read wrong in Polish, Russian or Czech for 2-4 or 5+.

**Cause:** `x_one` / `x_other` string pairs, or one flat `"%1$d days"` string, assume English categories. A translator cannot add a category that the key does not have.

**Fix:** use `<plurals>` for every count, and give each locale all its categories. Japanese uses only `other`. Polish needs `one`, `few`, `many`, `other`.

```xml
<plurals name="days">
    <item quantity="one">%1$d dzień</item>
    <item quantity="few">%1$d dni</item>
    <item quantity="many">%1$d dni</item>
    <item quantity="other">%1$d dnia</item>
</plurals>
```
```kotlin
pluralStringResource(Res.plurals.days, count, count) // quantity, then format args
```
- One count per plural. A sentence with two numbers becomes two plurals.
- Verify with 1, 3 and 16 on one screen: in Polish that shows `one`, `few` and `many`.

**Seen on:** CMP 1.7.3 and 1.11.1.

## 8. Missing keys fall back one key at a time, and the compiler cannot see it

**Symptom:** a device in one language shows a mix of that language and the default. One locale drops an argument or a plural form. The build is green.

**Cause:** resolution is per key. A key missing from `values-es` falls back to `values/` for that key only. The Kotlin compiler checks only that an accessor exists. It never compares one locale file with another.

**Fix:** add a check that reads the XML files (any script language works) and run it before merge:
- exact key-set parity with `values/`, and no key that is `<string>` in one file and `<plurals>` in another
- the same set of positional placeholders per key and per plural category (reordering is legal, a drop is not)
- plural categories per locale pinned in a table, not read from `Intl.PluralRules`, which follows live CLDR and adds categories over time
- a floor on the number of parsed keys, so a reader that matches nothing fails instead of passing every check

**Seen on:** CMP 1.7.3 and 1.11.1.

## 9. Dates, numbers and currency: format with the platform, in the catalog's locale

**Symptom:** dates show English month names in every language. A translated sentence carries a date or an amount in a different format from the text around it, for example a decimal comma inside an English sentence on a device whose language has no catalog.

**Cause:**
- kotlinx-datetime has no locale-aware month or weekday names; its formats take caller-supplied names. Common Kotlin has no `String.format`, and `Double.toString()` always writes `.`.
- `uppercase()` / `lowercase()` without a locale use invariant rules, which are wrong for Turkish `i`.
- Platform formatters default to the device locale. CMP falls back to `values/` when the device language has no folder, so the formatter and the text can disagree.

**Fix:**
- Put a key in each catalog whose value is that catalog's own tag (`<string name="catalog_locale">en</string>` in `values/`, `pl` in `values-pl/`). Read it and pass it to every formatter.
- Wrap the platform formatters in expect/actual: `DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM)` / `NSDateFormatter` (medium style) for dates, `NumberFormat` / `NSNumberFormatter` for numbers and currency.
- On iOS, build the `NSDate` for a calendar day at local noon, so a DST change at midnight cannot move it to the day before.
- Store all-caps labels already uppercased in each catalog.

```kotlin
expect fun formatMoney(amount: Double, currency: String, localeTag: String): String
// androidMain
actual fun formatMoney(amount: Double, currency: String, localeTag: String): String =
    NumberFormat.getCurrencyInstance(Locale.forLanguageTag(localeTag))
        .apply { this.currency = Currency.getInstance(currency) }
        .format(amount)
// iosMain
actual fun formatMoney(amount: Double, currency: String, localeTag: String): String =
    NSNumberFormatter().apply {
        numberStyle = NSNumberFormatterCurrencyStyle
        locale = NSLocale(localeIdentifier = localeTag)
        currencyCode = currency
    }.stringFromNumber(NSNumber(double = amount)) ?: amount.toString()
```

**Seen on:** kotlinx-datetime 0.6.1 and 0.7.1, CMP 1.11.1, Kotlin 2.4.10.

## 10. If your code reads the device language: use the right API

Applies if you read the language yourself, for example to send it to a backend or to pick content. CMP picks its own resource folder.

**Symptom:** users whose phone region and phone language differ get the wrong language. Hebrew or Indonesian users fall back to the default.

**Cause:** on iOS, `NSLocale.currentLocale` carries the Region setting and the app's active localization; the user's language list is `NSLocale.preferredLanguages`. On Android, `Locale.getDefault().language` can return legacy ISO codes (`iw` for Hebrew, `in` for Indonesian).

**Fix:**
- iOS: read `NSLocale.preferredLanguages.firstOrNull()` and strip subtags when you need a bare code. CMP 1.11.1 also reads `preferredLanguages` first to pick resources.
- Android: map legacy codes to the current ones before any lookup.

**Seen on:** CMP 1.7.3 and 1.11.1, iOS 18, Android minSdk 26.

## Related

- See kmp-testing-strategy: JVM tests and Compose strings.
- See kmp-api-contracts: kotlinx decode of unknown enum values.
- See kmp-agent-device-testing: per-app locale on a device.

## Checklist

- [ ] No `getString` from a Main-dispatched coroutine; one `withContext(Dispatchers.Default)` helper.
- [ ] Push loc-keys and shortcut labels mirrored into Android `res/`, placeholder-free, names pinned.
- [ ] iOS `Localizable.strings` / `InfoPlist.strings` per locale, in the Resources phase, checked in the built `.app`.
- [ ] No `\'` or `\"` in composeResources.
- [ ] Only `%1$s` / `%1$d`; numbers formatted before substitution.
- [ ] Language-only folders, region only in the text or the `.lproj` name.
- [ ] Every count is a `<plurals>` with all the locale's categories.
- [ ] Parity check for keys, placeholders and plural categories runs before merge.
- [ ] Dates, numbers and currency formatted by the platform, with the catalog's locale tag.
- [ ] iOS language from `preferredLanguages`; Android legacy codes mapped.
