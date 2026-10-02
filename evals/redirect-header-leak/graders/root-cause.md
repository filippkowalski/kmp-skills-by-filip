---
type: llm
weight: 2
focus: last_message
---
Context: a Ktor 3.1 client adds `Authorization: Bearer ...` and a custom `X-Device-Attestation` header in an `HttpSend` interceptor, only when `request.url.host` equals the API host. An API endpoint now answers 302 with a `Location` on a CDN on another host. The client keeps Ktor's default redirect handling. The developer asks whether anything reaches the CDN.

PASS if the answer states that the `X-Device-Attestation` header (the attestation token) is sent to the CDN host, and explains why:
- Ktor follows the redirect itself (the `HttpRedirect` plugin, on by default through `followRedirects = true`). It builds the follow-up request from the original request and copies its headers.
- On a redirect to another host (authority), it removes only the `Authorization` header. A custom header stays.
- The host check only decides whether to ADD headers. It never removes a header that is already on the copied request.
Saying that the bearer token is stripped on the cross-host redirect is expected. An answer that wrongly says the bearer token also leaks can still pass, but only if it names the attestation header leak and the header-copy mechanism.

FAIL if any of these is true:
- It says the interceptor is safe because the host check keeps the headers off other hosts.
- Its only finding is that the bearer token leaks, with no mention of the attestation header.
- It says nothing reaches the CDN because the interceptor runs again for the redirected request and skips it.
- It only suggests hardening the host comparison (subdomains, `endsWith`, scheme, port), certificate pinning, or moving auth to the Auth plugin.
- It gives a generic "redirects can leak headers" without saying that the attestation header reaches the CDN and how the headers get there.
