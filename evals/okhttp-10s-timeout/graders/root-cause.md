---
type: llm
weight: 2
focus: last_message
---
Context: a Ktor 3.1 client sets `HttpTimeout { requestTimeoutMillis = 60_000; connectTimeoutMillis = 15_000 }`. Raising both did not help. A report endpoint that sends nothing for 20 to 40 s and then answers fails at about 10 s on Android (OkHttp engine) with SocketTimeoutException. iOS (Darwin engine) works. A long streaming download on Android works.

PASS if the answer states both of these facts:
- The 10 s is OkHttp's own default read timeout (a socket or idle timeout: no bytes for 10 s), not a limit on the whole call.
- The Ktor OkHttp engine sets OkHttp's read (and write) timeout only from `socketTimeoutMillis`. This client never sets `socketTimeoutMillis`, so OkHttp keeps its 10 s default, and a longer `requestTimeoutMillis` or `connectTimeoutMillis` cannot extend it.
The answer may also say that iOS works because the Darwin engine (NSURLSession) uses a longer default (60 s), and that the export passes because bytes keep arriving. These are expected but not required.

FAIL if the answer's explanation is any of these:
- `requestTimeoutMillis` is still too low, or the fix is only to raise it further.
- The server, load balancer, proxy, CDN or the Android emulator network cuts the call.
- The connect timeout, DNS or the TLS handshake.
- Android or OkHttp caps the whole request at 10 s (the 50 s download contradicts this).
- A vague "check your timeout configuration" or "OkHttp has different defaults" that never names OkHttp's read timeout and `socketTimeoutMillis`.
- `socketTimeoutMillis` appears only as one of several unranked guesses.
