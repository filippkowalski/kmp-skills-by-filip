---
type: llm
weight: 2
focus: last_message
---
Context: a kotlinx.serialization DTO has `val status: OrderStatus` (a plain `@Serializable` enum, no default, not nullable). The `Json` has `ignoreUnknownKeys = true` and `coerceInputValues = true`. The server sent a new status value, and the whole `List<OrderDto>` failed to decode. The developer asks why `coerceInputValues` did not help.

PASS if the answer states all of these:
- kotlinx decodes the enum by `@SerialName` and throws a `SerializationException` for a value it does not know.
- `coerceInputValues` replaces an unknown enum value only when the property has a default value (or, with `explicitNulls = false`, when the property is nullable). `status` has no default and is not nullable, so decoding still throws.
- The exception aborts the whole response decode, so every order fails, not only the pickup order.

FAIL if any of these is true:
- It blames `ignoreUnknownKeys` or says an option is missing, such as `isLenient`.
- It says `coerceInputValues` works only for null values, or does not apply to enums.
- Its main explanation is that the `Json` instance is not wired into ContentNegotiation.
- It never states the default-value (or nullable) condition of `coerceInputValues`.
- It gives only a generic "the API contract changed, version your API" with no decode mechanism.
