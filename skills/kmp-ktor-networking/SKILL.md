---
name: kmp-ktor-networking
description: Ktor client behaviour that bites Kotlin Multiplatform apps on Android (OkHttp engine) and iOS (Darwin engine). Use when you configure HttpClient, HttpTimeout, WebSockets or SSE, interceptors, attestation headers or bearer auth, when you show network errors to users, when you catch exceptions around suspend calls, or when a slow request fails after about 10 seconds on Android only. Triggers - ktor, okhttp, darwin engine, HttpTimeout, requestTimeoutMillis, socketTimeoutMillis, SocketTimeoutException, HttpRequestTimeoutException, webSocketSession, HttpSend, intercept, Auth plugin, bearer, refreshTokens, App Check, runCatching, CancellationException, expectSuccess, non-2xx, ktor-bom, ktor version.
---

# Ktor networking in KMP

Library and platform behaviour only. The most expensive trap is first.

## 1. The OkHttp engine keeps a 10 s idle read timeout unless socketTimeoutMillis is set

- **Symptom:** An endpoint that thinks for more than 10 s before it answers fails on Android.
  iOS works. A request that asks for `requestTimeoutMillis = 30_000` still fails at 10 s.
- **Cause:** The Ktor OkHttp engine sets OkHttp's read and write timeouts only from
  `socketTimeoutMillis`. When it is unset, OkHttp keeps its own 10 s default. It is an idle
  timeout (no bytes for 10 s), not a cap on total time: a streaming download that keeps sending
  passes. A longer `requestTimeoutMillis` cannot extend it. The Darwin engine uses the 60 s
  NSURLSession request default, so iOS hides the bug.
- **Fix:** Set both values together, per request, through one helper. If the same client also
  runs WebSockets or SSE, keep only the connect timeout client-wide, so request and socket
  timeouts never hit a long-lived connection that is idle between messages.

```kotlin
val client = HttpClient {
    install(WebSockets)
    install(HttpTimeout) { connectTimeoutMillis = 15_000 } // only this, if sockets share the client
}

fun HttpRequestBuilder.timeBudget(millis: Long) = timeout {
    requestTimeoutMillis = millis
    socketTimeoutMillis = millis // without this, OkHttp keeps its 10 s read timeout
}

client.get("$base/report") { timeBudget(60_000) }
client.get("$base/row") { timeBudget(10_000) } // fast calls too: a decision, not an engine default
```

- **Client timeout vs server work:** when the server can take longer than the client waits, the
  server can finish a write after the client gave up, and a retry writes it twice. Set the budget
  above the server's own timeout for that route. For retryable writes, send an idempotency key
  made once per user action (not per attempt), and keep it until a final answer arrives.
- **Seen on:** ktor-client-okhttp 3.0.3 (decompiled `setupTimeoutAttributes`), still applied on
  3.1.0; ktor-client-darwin 3.0.3 and 3.1.0.

## 2. Ktor exception messages contain the full request URL

- **Symptom:** An error banner shows your API host, the route, and ids from the path.
- **Cause:** Ktor timeout exceptions put the URL in the message
  (`Socket timeout has expired [url=..., socket_timeout=...]`). If a path holds a user id, the
  message holds personal data. It hides well inside a formatted string:
  `getString(Res.string.error_x, error.message)` looks safe and is not.
- **Fix:** Deny by default. Release builds show localized copy chosen from a stable error code,
  else a generic line. Only debug builds show raw text. Route every error string through one
  helper; an allow-list of "safe" exception types is never complete.

```kotlin
suspend fun userFacingError(error: Throwable): String =
    (error as? ApiError)?.code?.let { localizedForCode(it) }
        ?: if (isDebugBuild) error.message ?: genericError() else genericError()
```

- **Seen on:** Ktor 3.0.3, 3.1.0 (OkHttp and Darwin).

## 3. runCatching catches CancellationException

- **Symptom:** A cancelled load reports success from a fallback or cache. A closed screen paints
  an error. One failed child in `coroutineScope` does not stop its siblings. A
  `withTimeoutOrNull` deadline turns into one more pass of a retry loop.
- **Cause:** `runCatching` and `catch (e: Throwable)` catch `CancellationException` like any
  other exception, so cancellation stops propagating.
- **Fix:** Use a suspend-safe variant, and rethrow cancellation first in every broad catch.

```kotlin
suspend inline fun <T> runSuspendCatching(block: () -> T): Result<T> = try {
    Result.success(block())
} catch (e: CancellationException) {
    throw e
} catch (e: Throwable) {
    Result.failure(e)
}
```

Grep for `runCatching {` around suspend calls before each release.
- **Seen on:** kotlinx-coroutines 1.10, Kotlin 2.1 to 2.4, all targets.

## 4. Ktor does not throw on non-2xx responses by default

- **Symptom:** A write that got 401 or 500 is dropped with no error. Or a 500 body fails as a
  serialization error with a misleading message.
- **Cause:** `expectSuccess` is false by default, so `post()` returns normally on any status.
- **Fix:** Check the status on every call, or set `expectSuccess = true` and handle
  `ResponseException`. Parse the error body into a typed exception with a stable code. Give codes
  that change the UI (upgrade, wait and retry) their own subclass, so callers `catch` a type
  instead of comparing strings. Use the same mapper for WebSocket error frames.

```kotlin
suspend inline fun <reified T> HttpResponse.bodyOrApiError(): T {
    if (!status.isSuccess()) throw apiFailure(status, bodyAsText()) // code + args + debug message
    return body()
}
```

- **Seen on:** Ktor 3.0.3, 3.1.0.

## 5. Every request, WebSocket upgrades included, runs through HttpSend interceptors

- **Symptom:** A new call or a socket goes out without your header. Or a credential goes to
  another host, because another library (for example Coil `coil-network-ktor3`) shares the client
  or a redirect points elsewhere.
- **Cause:** All requests on an `HttpClient` pass through its `HttpSend` interceptors: REST calls,
  WebSocket upgrades, and requests that other libraries build on it.
- **Fix:** Attach per-request headers in one interceptor, scoped to your host. Compare the host,
  not the origin, because upgrades use `wss`. Then split by kind of header:
  - **If you send an attestation header** (Firebase App Check or similar): fail soft. Cap the
    wait, send no header on error, rethrow cancellation.
  - **Auth tokens:** fail closed. Use the Ktor `Auth` plugin (`bearer { loadTokens {};
    refreshTokens {} }`). After a 401 it refreshes once for all failed requests and retries them.
    Its `sendWithoutRequest` defaults to `{ true }`: the token goes on the first request to any
    host. Scope it with `sendWithoutRequest { it.url.host == apiHost }`.
- **A host check does not cover redirects.** `HttpRedirect` (on by default) copies the request
  headers to the redirect target and, on a cross-authority redirect, removes only
  `Authorization`. A custom attestation or secret header follows the redirect to the other host.
  On a client that adds such a header, set `followRedirects = false` and handle 3xx yourself, or
  check the redirect target and strip the header before the request leaves your host.

```kotlin
client.plugin(HttpSend).intercept { request ->
    if (request.url.host.equals(apiHost, ignoreCase = true)) {
        attestationToken()?.let { request.headers.append("X-Firebase-AppCheck", it) }
    }
    execute(request)
}

private suspend fun attestationToken(): String? = withTimeoutOrNull(3_000) {
    try { provider() } catch (e: CancellationException) { throw e } catch (_: Throwable) { null }
}?.trim()?.takeIf { it.isNotEmpty() }
```

- **If you use App Attest on iOS:** `AppAttestProvider(app:)` never returns nil, even on a
  simulator; each token fetch fails later. Check `DCAppAttestService.shared.isSupported` first.
  Fail-soft hides a broken provider, so log a breadcrumb before your server enforces attestation.
- **Seen on:** Ktor 3.0.3 and 3.1.0 (`HttpRedirect` and `BearerAuthConfig` read at 3.1.0), Coil
  3.5.0, Firebase App Check (Android BOM 33.7.0, iOS SDK).

## 6. If you use WebSockets, send the token in the upgrade header, not the URL

- **Symptom:** Bearer tokens appear in proxy logs, server access logs, and bug reports that
  include the URL.
- **Cause:** Browser WebSocket APIs cannot set headers, so many examples pass the token as a
  query parameter. The Ktor client can set headers on the upgrade, on OkHttp and Darwin.
- **Fix:** Put the token in the `Authorization` header of the upgrade. Open every socket through
  one helper, so a new socket cannot forget it.

```kotlin
suspend fun openSocket(path: String) =
    client.webSocketSession("$wsBase$path") { header(HttpHeaders.Authorization, bearer()) }
```

- **Seen on:** Ktor 3.1.0, OkHttp and Darwin engines.

## 7. If you use WebSockets, cancel the session after a graceful close

- **Symptom:** A socket stays open after a timed-out close, or teardown takes longer than its budget.
- **Cause:** The graceful `session.close()` suspends. When the caller's deadline cancels it, the
  session itself is not cancelled, so the transport stays open.
- **Fix:** `try { session.close() } finally { session.cancel() }`. Do not wrap `close()` in
  `runCatching` (see rule 3).
- **Seen on:** Ktor 3.1.0 (`DefaultClientWebSocketSession`), OkHttp and Darwin.

## 8. Ktor artifacts version as one set

- **Symptom:** Runtime failures or odd behaviour after you add a library that depends on Ktor.
- **Cause:** A transitive dependency can make Gradle raise `ktor-client-core` alone (for example
  `coil-network-ktor3` 3.5.0 requires 3.1.0), while engines and plugins stay on the old version.
- **Fix:** Use one version for every `io.ktor` artifact, test artifacts included
  (`ktor-client-mock`), and bump them together. In KMP source sets, the Ktor docs use one version
  in `libs.versions.toml`; in an Android-only or JVM module, `platform("io.ktor:ktor-bom:<v>")`
  does the same. Read the resolved versions after you add a Ktor-based library.
- **Seen on:** Ktor 3.0.3 to 3.1.0, Coil 3.5.0.

## Owned elsewhere

- See kmp-app-architecture: ownership checks after suspension, merge results onto live state.
- See kmp-api-contracts: unknown enum values in responses, `encodeDefaults`, nullable and default fields.
- See kmp-ios-build: Swift bridge callbacks (a token provider written in Swift).

## Checklist

- [ ] Every call sets request AND socket timeout; with sockets on the client, only connect is client-wide.
- [ ] Client budgets sit above the server's timeout; retryable writes carry one key per action.
- [ ] No raw `error.message` in release UI, also not inside a formatted string.
- [ ] No `runCatching` or `catch (Throwable)` around suspend calls without a cancellation rethrow.
- [ ] Every response status is checked; errors become typed exceptions with stable codes.
- [ ] One host-scoped `HttpSend` interceptor; auth tokens fail closed; bearer `sendWithoutRequest` scoped to your host.
- [ ] If you use app attestation, the header fails soft (capped wait, no header on error).
- [ ] A client that adds a custom secret header sets `followRedirects = false` or strips it on redirects.
- [ ] WebSocket tokens go in the upgrade header; close cancels the session in `finally`.
- [ ] All `io.ktor` artifacts share one version.
- [ ] Test Android against an endpoint slower than 10 s before you trust iOS results.
