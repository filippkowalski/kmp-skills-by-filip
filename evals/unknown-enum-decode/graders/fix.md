---
type: llm
weight: 1
focus: last_message
---
Context: `OrderDto.status` is a plain kotlinx enum with no default. A new server value fails the whole order list decode. The developer wants a client-side model that survives the next new status.

PASS if the fix gives the status an unknown fallback on the client, by one of these:
- Carry the status as a `String` on the wire and map it in one total `when` with an `else` branch that degrades (for example to `Unknown` or a generic "processing" state) and logs.
- A custom `KSerializer` for the enum that maps unknown strings to an `Unknown` entry.
- Add an `Unknown` entry and give the property a default (`val status: OrderStatus = OrderStatus.Unknown`) so the existing `coerceInputValues = true` applies, or make it nullable with `explicitNulls = false`.
The order with the unknown status must still show (with a generic status), not disappear.

FAIL if any of these is true:
- The only fix is on the server (hide values, API versioning).
- It rewrites the JSON in a pre-pass before decoding.
- It catches the decode error and shows an empty list or a cached list.
- It decodes items one by one and drops the orders that fail, so the user loses those orders.
- It relies on `isLenient`, `ignoreUnknownKeys` or `@JsonNames` alone.
- It adds harmful advice, such as catching `CancellationException` or a blanket `runCatching` around the suspend call.
