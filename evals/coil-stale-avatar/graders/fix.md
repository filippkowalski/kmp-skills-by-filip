---
type: llm
weight: 1
focus: last_message
---
Context: Coil 3.5.0 with `KtorNetworkFetcherFactory()` serves avatars from the disk cache forever, although the CDN sends revalidation headers. The URL cannot change and there is no version field.

PASS if the fix adds the `io.coil-kt.coil3:coil-network-cache-control` artifact and passes `CacheControlCacheStrategy()` to the fetcher through the lambda overload, for example `KtorNetworkFetcherFactory(httpClient = { client }, cacheStrategy = { CacheControlCacheStrategy() })` (with the `ExperimentalCoilApi` opt-in), so Coil revalidates disk hits with the ETag. Also removing the uploader's own disk and memory entries after an upload is a fine extra.

FAIL if any of these is true:
- The only fix is removing cache entries on the uploader's device (other devices stay stale).
- It disables the disk cache or network caching (`diskCachePolicy(DISABLED)` and similar).
- It appends a random or timestamp query parameter to each load, or clears the whole cache on every launch.
- It installs Ktor's `HttpCache` plugin as the fix.
- It adds harmful advice, such as disabling TLS checks.
