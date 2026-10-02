---
type: llm
weight: 1
focus: last_message
---
Context: a Ktor 3.1 client with the OkHttp engine sets only `requestTimeoutMillis` and `connectTimeoutMillis`. A report endpoint that is silent for 20 to 40 s fails at 10 s on Android. The client has no WebSockets or SSE.

PASS if the fix sets `socketTimeoutMillis` together with `requestTimeoutMillis`, to a budget that covers the slow report (for example 60_000 or more). The best form is per request, `client.get(...) { timeout { requestTimeoutMillis = 60_000; socketTimeoutMillis = 60_000 } }`, ideally through one helper that every call uses. Setting both values in the client-wide `HttpTimeout` block also passes, because this client has no long-lived sockets. An equally correct alternative: set OkHttp's own `readTimeout` in the engine config (`engine { config { readTimeout(...) } }`) to a budget that covers the report.

FAIL if any of these is true:
- The fix only raises `requestTimeoutMillis` and/or `connectTimeoutMillis`.
- It removes all timeouts (infinite or 0 for the socket or read timeout on the whole client).
- Its main fix is to retry after the timeout, or to switch the Android engine.
- It adds harmful advice: disabling TLS or certificate checks, catching `CancellationException`, or wrapping suspend calls in plain `runCatching` without rethrowing cancellation.
