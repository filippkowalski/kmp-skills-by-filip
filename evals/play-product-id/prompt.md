---
name: play-product-id
tags: [kmp-revenuecat]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "The paywall is empty on Android only and the backend ignores Android purchases, because Play product ids arrive as subscriptionId:basePlanId while the maps use the App Store form."
expected_outcome: "Explains that RevenueCat reports Play products as subscriptionId:basePlanId (fit.pro.monthly:monthly) in StoreProduct.id and webhooks, and maps each plan from both store forms on the client and the backend."
---
Our fitness app sells Pro monthly and annual through RevenueCat (purchases-kmp-core 3.5.0, Kotlin 2.4.10, CMP 1.11.1). iOS works end to end. On Android the paywall shows no plans at all, only the empty state.

What I checked: the Play subscriptions are active (subscription `fit.pro.monthly` with base plan `monthly`, and `fit.pro.annual` with base plan `annual`). The build comes from the internal testing track, and the tester is a license tester. The offering loads on Android; our log prints `offering=default packages=2 ($rc_monthly 4,99 €, $rc_annual 39,99 €)`. So the prices arrive. Also, our backend grants Pro from RevenueCat webhooks, and it ignored the one Android test purchase we made from an older build.

```kotlin
enum class Plan { Monthly, Annual }

private val planByProductId = mapOf(
    "fit.pro.monthly" to Plan.Monthly,
    "fit.pro.annual" to Plan.Annual,
)

suspend fun paywallRows(): List<PaywallRow> {
    val offering = Purchases.sharedInstance.awaitOfferings().current ?: return emptyList()
    return offering.availablePackages.mapNotNull { pkg ->
        val plan = planByProductId[pkg.storeProduct.id] ?: return@mapNotNull null
        PaywallRow(plan = plan, price = pkg.storeProduct.price.formatted, pkg = pkg)
    }
}
```

```ts
// backend webhook handler (Node)
const PLAN_BY_PRODUCT = { "fit.pro.monthly": "monthly", "fit.pro.annual": "annual" };
const plan = PLAN_BY_PRODUCT[event.product_id]; if (!plan) return res.sendStatus(200);
```

Why does Android drop both packages when the prices are clearly there, and what is the fix on both sides?
