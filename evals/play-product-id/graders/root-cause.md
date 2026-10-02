---
type: llm
weight: 2
focus: last_message
---
Context: a purchases-kmp 3.5.0 paywall maps `pkg.storeProduct.id` to a plan with keys `fit.pro.monthly` and `fit.pro.annual`. On iOS it works. On Android the offering loads with two priced packages, but the paywall is empty. The Play subscriptions are `fit.pro.monthly` (base plan `monthly`) and `fit.pro.annual` (base plan `annual`). A backend maps the webhook `product_id` with the same keys and ignored an Android purchase.

PASS if the answer states that on Google Play, RevenueCat reports the product id as `subscriptionId:basePlanId` (here `fit.pro.monthly:monthly` and `fit.pro.annual:annual`) in `StoreProduct.id`, in customer subscription keys and in the webhook `product_id`, while the App Store uses the plain id. So the client map misses both Android packages (both rows are dropped), and the backend map misses the Android webhook the same way.

FAIL if any of these is true:
- It blames inactive products, propagation delay, license testers, the testing track or the application id (the offering and prices load).
- It blames the offering configuration or price formatting.
- It says "product ids differ per store" without the `subscriptionId:basePlanId` form.
- It blames a missing `getProducts` call or purchase acknowledgement.
- It names the composite id only as one of several unranked guesses.
