---
name: kmp-revenuecat
description: RevenueCat purchases-kmp and App Store / Google Play behaviour that bites Kotlin Multiplatform apps. Only for apps that sell subscriptions or in-app purchases through RevenueCat. Use when you add or debug paywall prices, offerings, packages, purchase errors, restore, trial copy, or purchase testing on the iOS simulator or an Android testing-track build. Triggers - revenuecat, purchases-kmp, offering, Package, PackageType, StoreProduct, base plan, subscriptionId:basePlanId, awaitPurchase, PurchasesErrorCode, restore purchases, restore behavior, transfer, AlreadyOwned, introductoryDiscount, trial eligibility, lifetime, StoreKit configuration, Play Billing testing track, license tester, Test Store, RevenueCat API key.
---

# RevenueCat in KMP

purchases-kmp, Play Billing and StoreKit behaviour only, for apps that sell through RevenueCat.
The most expensive trap is first.

## 1. If you sell on both stores, one subscription has a different product id on each

- **Symptom:** Everything works on iOS. On Android, prices are missing, the buy button is dead,
  and any backend that maps product ids to plans rejects or ignores every Android purchase.
- **Cause:** The App Store names a subscription `app.sub.monthly`. Google Play names the billable
  item `app.sub.monthly:monthly` (`subscriptionId:basePlanId`). RevenueCat passes each store's form
  through verbatim: in `StoreProduct.id`, in customer subscription keys and in webhook product ids.
- **Fix:** Map each plan to one id per store, everywhere you map products to plans (client and
  backend). Resolve a plan from either form. Key UI state by your plan id, never by a product id.
  Test both forms; the iOS simulator never shows the bug.

```kotlin
object BillingCatalog {
    private val idsByPlan = mapOf(
        "monthly" to listOf("app.sub.monthly", "app.sub.monthly:monthly"),
        "annual" to listOf("app.sub.annual", "app.sub.annual:annual"),
    )
    fun planFor(storeProductId: String): String? =
        idsByPlan.entries.firstOrNull { storeProductId in it.value }?.key
}
```

- **Seen on:** purchases-kmp 2.10.2 and 3.5.0, Play Billing through RevenueCat.

See also: rc-subscriptions in RevenueCat/ai-toolkit (`activeSubscriptions` entries are `subscriptionId:basePlanId`).

## 2. Buy the offering Package; Play product queries take the subscription id only

- **Symptom:** If you sell on Google Play, the purchase sheet never opens on Android.
- **Cause:** Play Billing product queries accept the subscription id only. `getProducts` with
  `sub:basePlan` returns nothing, but a product it does return reports the composite id.
- **Fix:** Purchase the `Package` from the current offering (`awaitPurchase(pkg)`). This skips the
  id query, keeps RevenueCat offering metrics and experiments correct, and on Play charges the
  subscription option the row was priced from. If you must query by id, query
  `id.substringBefore(':')`, then select the result by the full id.
- **Seen on:** purchases-kmp 2.10.2 (`awaitGetProducts`), 3.5.0 (`awaitPurchase(Package)`).

See also: revenuecat-purchase-flow in RevenueCat/ai-toolkit (the offering-to-purchase flow, with a KMP platform file).

## 3. A package type does not guarantee the product's billing period

- **Symptom:** A row labelled "Annual" bills monthly, or a price-test product your backend does
  not know gets sold and never unlocks.
- **Cause:** The RevenueCat dashboard lets you attach any product to any package type.
- **Fix:** Before a package becomes a row, check that the package type and the product's own
  `period` agree, and that you know the product id. Drop and log a package that disagrees. Label
  rows from the product's period, and compare periods by value (7 days equals 1 week), not by
  string. Keep lifetime and one-time products: their `period` is null, which is correct for a
  `LIFETIME` package, and `CUSTOM` packages imply no period at all.

```kotlin
val sellable = offerings.current?.availablePackages.orEmpty().filter { pkg ->
    val periodOk = when (pkg.packageType) {
        PackageType.CUSTOM, PackageType.UNKNOWN -> true              // type implies no period
        else -> pkg.storeProduct.period?.normalized() ==             // LIFETIME: both null
            pkg.packageType.expectedPeriod()                         // e.g. MONTHLY -> 1 month
    }
    (periodOk && BillingCatalog.planFor(pkg.storeProduct.id) != null)
        .also { if (!it) log("drop ${pkg.identifier}") }
}
```

- **Seen on:** purchases-kmp 3.5.0.

## 4. A failed logIn leaves the SDK on the previous app user

- **Symptom:** A card is charged under the previous user id after an account switch. The new
  account has no access.
- **Cause:** When `logIn` fails, purchases-kmp keeps the current `appUserID`. A purchase then runs
  under the old identity.
- **Fix:** `configure` once, then `logIn` only when `appUserID` differs. Call that idempotent login
  again right before `purchase`, and abort with a retryable message if it fails.

```kotlin
suspend fun login(userId: String) = runSuspendCatching { // see kmp-ktor-networking rule 3
    if (!Purchases.isConfigured) Purchases.configure(PurchasesConfiguration(key) { appUserId = userId })
    else if (Purchases.sharedInstance.appUserID != userId) Purchases.sharedInstance.awaitLogIn(userId)
}
// before the sheet: if (login(currentUser).isFailure) return showRetryableError()
```

- **Seen on:** purchases-kmp 2.10.2, 3.5.0. For the user check after the purchase returns, see
  kmp-app-architecture: ownership checks after suspension.

## 5. Purchase results arrive as exceptions: rethrow cancellation, then map the code in common code

- **Symptom:** A user who closes the sheet sees an error. A pending or already-owned purchase
  shows as a failure. A cancelled coroutine reports a purchase error instead of stopping.
- **Cause:** `awaitPurchase` returns only on success. Cancel, pending and already-owned arrive as
  `PurchasesTransactionException` (a `PurchasesException` with `userCancelled`). A broad `catch`
  or `runCatching` around the suspend call also catches coroutine `CancellationException`.
- **Fix:** In common code, rethrow `CancellationException` first, then check `userCancelled`,
  then map `code` to a small outcome type the UI can act on (cancelled, pending, already owned,
  retryable, failed). RevenueCat's KMP samples wrap suspend calls in plain `runCatching`; add the
  rethrow before you reuse them (see kmp-ktor-networking rule 3).
- **Seen on:** purchases-kmp 2.10.2, 3.5.0.

See also: rc-error-handling in RevenueCat/ai-toolkit (the per-code handling table and which exception each call throws).

## 6. If you use anonymous sign-in and a backend that mirrors entitlements, sync it after restore

- **Symptom:** After a reinstall or on a new device, Restore reports success and nothing unlocks.
- **Cause:** Anonymous sign-in (for example Firebase anonymous auth) creates a new user id on a
  new device and on an Android reinstall without Auto Backup (iOS keeps it in the Keychain: see kmp-ios-build rule 10).
  The store account still owns the purchase, and what a restore does with it depends on the
  project's restore behaviour setting in RevenueCat. The default, "Transfer to new App User
  ID", moves it to the current user; "Keep with original App User ID" returns an error instead.
  A backend keyed on your user id has no record for the new id.
- **Fix:** Check the project's restore behaviour setting in the RevenueCat dashboard and design
  for it. With transfer, after `awaitRestore` (and after an `AlreadyOwned` purchase result), ask
  your backend to re-check the store account for the current user id, and report success only
  when the backend confirms access.
- **Seen on:** purchases-kmp 2.10.2, 3.5.0, Firebase anonymous auth.

## 7. Prices and trials come from the offering, and trial eligibility is per user

- Render the rows the current offering lists, in its order, with `price.formatted`. When the
  offering is empty or fails, show an empty state with a retry. Do not fall back to prices
  compiled into the app.
- A trial line must state its length: `introductoryDiscount.subscriptionPeriod` in days times
  `numberOfPeriods` ("3 x 1 week" is 21 days). If the period cannot be read exactly, drop the line.
- `introductoryDiscount` describes the product, not the user. On iOS, call
  `awaitTrialOrIntroPriceEligibility(products)` before you promise a trial. On Android the call
  returns `UNKNOWN` for every product and logs "only available on iOS"; treat `UNKNOWN` as "no
  claim", not as eligible.
- **Seen on:** purchases-kmp 3.5.0 (eligibility behaviour read from the Android classes).

See also: rc-subscriptions in RevenueCat/ai-toolkit (Google Play returns only the offers a user is eligible for).

## 8. Keys and local testing

- Put the SDK behind an interface (`isAvailable`, `login`, `currentOffering`, `purchase`,
  `restore`). With no public SDK key it reports unavailable, so a fresh clone still builds.
- Read the Android key from `local.properties` or an env var into `BuildConfig`, the iOS key from
  `Info.plist`. Fail the release build when the key is blank: see kmp-store-release rule 2.
- Play Billing: the installed `applicationId` must match the Play app whose products are
  active. Real test purchases need a license-tester account and a signed build installed from a
  Play testing track. RevenueCat's testing guide: "A sideloaded debug build with a debug signing
  key cannot purchase licensed products, even for license testers." For fast UI work in debug,
  use RevenueCat's Test Store key instead. See kmp-store-release rule 1.
- iOS simulator: add a StoreKit configuration file to the Debug scheme's run action. Without it,
  offerings load but every price is empty. Keep it out of Release.
- purchases-kmp 3.x embeds RevenueCat's Swift in its klib; remove any old PurchasesHybridCommon
  SPM package. See kmp-ios-build: iOS test binary link errors.
- **Seen on:** purchases-kmp 3.5.0, AGP 8.13, Xcode with XcodeGen.

See also: revenuecat-testing-setup in RevenueCat/ai-toolkit (Test Store keys, license testers and the Internal Testing track).

## Checklist

- [ ] Selling on both stores: every plan maps to an App Store id AND a Play `subscriptionId:basePlanId` id.
- [ ] Purchases use the offering `Package`; UI state is keyed by plan id.
- [ ] Packages whose type and period disagree are dropped; lifetime and custom packages are kept.
- [ ] Login is idempotent and runs again right before the purchase sheet.
- [ ] Cancel, pending and already-owned have their own outcomes; cancellation is rethrown.
- [ ] Restore behaviour setting checked; with a mirroring backend, restore re-checks the current user.
- [ ] No hardcoded price or plan list; trial copy states its length and checks eligibility on iOS.
- [ ] Release fails without the SDK key; real Play purchases tested from a testing-track install; StoreKit file in Debug only.
- [ ] Test on an Android build installed from a Play testing track with a license tester, not only the iOS simulator.
