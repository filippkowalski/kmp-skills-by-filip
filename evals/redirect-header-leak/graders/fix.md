---
type: llm
weight: 1
focus: last_message
---
Context: a Ktor 3.1 client adds a custom `X-Device-Attestation` header (and a bearer token) in a host-scoped `HttpSend` interceptor. Ktor's default redirect handling copies the attestation header onto a 302 follow-up request to a CDN on another host.

PASS if the fix stops the attestation header (and any other custom secret header) from leaving on the redirect, by one of these:
- Set `followRedirects = false` on this client and handle the 3xx yourself: read `Location` and fetch the CDN URL with a request that carries none of the API headers (for example a separate plain client).
- In the same interceptor, strip the header when the host is not the API host, for example `else { request.headers.remove("X-Device-Attestation") }` (removing `Authorization` there too is fine), so the redirected request leaves without it.
- Have the API return the signed URL in the response body and download it with a client that adds no API headers (equivalent to handling the redirect yourself).

FAIL if any of these is true:
- It only tightens the host comparison, adds certificate pinning, or moves the bearer token to the Auth plugin, and the attestation header still follows the redirect.
- It says no change is needed.
- It moves the attestation token into the URL or query string.
- It adds harmful advice: disabling TLS or certificate validation, trusting all hosts, or catching `CancellationException` broadly.
