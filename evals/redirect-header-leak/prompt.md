---
name: redirect-header-leak
tags: [kmp-ktor-networking]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "A host-scoped HttpSend interceptor adds a custom attestation header; a 302 to a CDN on another host carries it along, because Ktor's redirect copies headers and strips only Authorization."
expected_outcome: "Says the X-Device-Attestation header reaches the CDN (HttpRedirect copies the request headers and removes only Authorization on a cross-host redirect) and fixes it with followRedirects = false or by stripping the header for other hosts."
---
We are a small team building a banking app (KMP, Kotlin 2.4.10, CMP 1.11.1, Ktor 3.1.0, OkHttp engine on Android, Darwin on iOS). Every call to our API carries the user's bearer token and a device attestation token. Both are added in one `HttpSend` interceptor that only touches requests to our API host, because other code shares this client (Coil loads images through it).

Last week the backend changed `GET /v1/statements/{id}/pdf`. It now answers `302 Found` with a `Location` on our CDN vendor (`https://files.cdn-vendor.net/...`, a signed URL). Downloads work on both platforms.

Security wants a review of the interceptor before the next release. Their concern is that the bearer token could reach the CDN vendor. Here is the client:

```kotlin
private const val API_HOST = "api.example-bank.com"

val client = HttpClient(platformEngine()) {
    install(ContentNegotiation) { json(appJson) }
    install(HttpTimeout) { connectTimeoutMillis = 15_000 }
}.apply {
    plugin(HttpSend).intercept { request ->
        if (request.url.host.equals(API_HOST, ignoreCase = true)) {
            sessionStore.accessToken()?.let { request.headers[HttpHeaders.Authorization] = "Bearer $it" }
            attestation.tokenOrNull()?.let { request.headers["X-Device-Attestation"] = it }
        }
        execute(request)
    }
}

suspend fun statementPdf(id: String): ByteArray =
    client.get("https://$API_HOST/v1/statements/$id/pdf").body()
```

Is this safe as written? Does anything we attach for our API end up at the CDN? If there is a problem, what is the smallest correct change?
