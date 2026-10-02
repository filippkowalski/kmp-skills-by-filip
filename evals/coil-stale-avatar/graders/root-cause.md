---
type: llm
weight: 2
focus: last_message
---
Context: a KMP app uses Coil 3.5.0 with `KtorNetworkFetcherFactory()` (no arguments) and a disk cache. Avatars are overwritten at the same URL. The CDN sends `Cache-Control: max-age=0, must-revalidate` and an `ETag`. The old image shows forever on every device, even after a relaunch; only a reinstall helps. The web app is fine.

PASS if the answer states both of these:
- Coil 3's default network cache strategy serves a disk-cache hit as it is and never sends a conditional request. It does not honour `Cache-Control` or `ETag`.
- The disk cache survives restarts, so the old bytes at the unchanged URL keep being served. Removing the memory-cache entry does not touch the disk entry.
Saying that HTTP cache-header support lives in a separate artifact (`coil-network-cache-control`) is expected but not required.

FAIL if any of these is true:
- It blames only the memory cache.
- It blames the CDN, the server headers, or the OS or NSURLCache (the web app already shows the new photo).
- It says the fix is Ktor's `HttpCache` plugin (Coil answers from its own disk cache before any request is made).
- It says only that `AsyncImage` needs a new cache key or model, without the missing revalidation.
- It stays vague ("image caching issue") and never says the default strategy ignores cache headers.
