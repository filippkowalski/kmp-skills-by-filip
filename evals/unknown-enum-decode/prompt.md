---
name: unknown-enum-decode
tags: [kmp-api-contracts]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "A new server enum value fails the whole list decode although coerceInputValues is on; coercion needs a default or a nullable property."
expected_outcome: "Explains that coerceInputValues replaces an unknown enum value only on a property with a default (or nullable with explicitNulls = false), so the throw aborts the whole list, and gives the status an unknown fallback (string plus total when, custom serializer, or a default Unknown)."
---
Yesterday our backend started to return a new order status, `ready_for_pickup`, for click-and-collect orders. Since then, users on the current store version (1.8.2) who have at least one such order see "Couldn't load your orders" on the Orders tab. Not only the pickup order is missing: the whole list is gone. Users without pickup orders are fine.

I thought we were protected against this, because our Json is lenient about new things:

```kotlin
val appJson = Json {            // installed with ContentNegotiation { json(appJson) }
    ignoreUnknownKeys = true
    coerceInputValues = true
}

@Serializable
enum class OrderStatus {
    @SerialName("pending") Pending,
    @SerialName("paid") Paid,
    @SerialName("shipped") Shipped,
    @SerialName("delivered") Delivered,
    @SerialName("cancelled") Cancelled,
}

@Serializable
data class OrderDto(
    val id: String,
    val totalCents: Long,
    val status: OrderStatus,
    val placedAt: String,
)

suspend fun orders(): List<OrderDto> = client.get("v1/orders").body()
```

Versions: Kotlin 2.4.10, kotlinx-serialization-json 1.7.3, Ktor 3.1.0, CMP 1.11.1. The backend can hide the new status from 1.8.2 for now, so I mostly care about the app side. Why does `coerceInputValues` not cover this, and how should the model look so the next new status does not take the whole screen down again?
