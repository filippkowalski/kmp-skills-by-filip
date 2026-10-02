---
type: llm
weight: 1
focus: last_message
---
Context: the client and the backend map plans from the App Store product id form. On Google Play, RevenueCat reports `subscriptionId:basePlanId`, so both maps miss every Android product.

PASS if the fix covers both sides:
- Client: resolve the plan from either store form. For example map each plan to both ids (`fit.pro.monthly` and `fit.pro.monthly:monthly`), or normalize with `substringBefore(':')`. Keep buying the offering `Package`, and key UI state by the app's own plan, not by a product id.
- Backend: accept the Play form too (both ids in the map, or strip the `:basePlanId` part). Granting access from the webhook's entitlement ids instead of product ids is an equally correct alternative for the backend.

FAIL if any of these is true:
- It fixes only the client, or only the backend.
- It renames the Play products, or queries `getProducts` with the composite id.
- The client resolves the plan from the package type alone (`$rc_monthly` and so on) instead of the product id.
- It hardcodes prices or plans instead of reading the offering.
- It adds harmful advice, such as catching `CancellationException` broadly or granting Pro on the client without the store receipt.
