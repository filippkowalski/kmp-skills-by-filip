---
name: okhttp-10s-timeout
tags: [kmp-ktor-networking]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "A slow Ktor call fails at 10 s on Android only although requestTimeoutMillis is 60 s; the OkHttp engine keeps its 10 s read timeout unless socketTimeoutMillis is set."
expected_outcome: "Names OkHttp's default 10 s read (idle) timeout, which the Ktor OkHttp engine overrides only from socketTimeoutMillis, and sets socketTimeoutMillis together with requestTimeoutMillis for the slow call."
---
Our fitness app is KMP (Kotlin 2.4.10, CMP 1.11.1, Ktor 3.1.0). `platformEngine()` returns the OkHttp engine on Android and the Darwin engine on iOS. We added a yearly training report. The server works on it for 20 to 40 seconds and then sends the whole JSON at once.

On iOS it works every time. On Android it fails at almost exactly 10 seconds with `io.ktor.client.network.sockets.SocketTimeoutException`. What confuses me: the same Android build downloads a 90 MB workout export in about 50 seconds with no error, so Android does not seem to cap the whole call at 10 seconds.

```kotlin
val client = HttpClient(platformEngine()) {
    expectSuccess = true
    install(ContentNegotiation) { json(appJson) }
    install(HttpTimeout) {
        requestTimeoutMillis = 60_000
        connectTimeoutMillis = 15_000
    }
    defaultRequest { url("https://api.example-fitness.com/") }
}

suspend fun yearlyReport(year: Int): ReportDto =
    client.get("v1/reports/yearly") { parameter("year", year) }.body()
```

I already raised `requestTimeoutMillis` to 120_000 and `connectTimeoutMillis` to 30_000. Still 10 seconds. The backend team says their load balancer allows 120 seconds, and iOS shows that the server does answer. Is this an OkHttp bug on Android? What should our timeout setup look like?
