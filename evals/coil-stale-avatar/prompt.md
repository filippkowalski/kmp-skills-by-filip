---
name: coil-stale-avatar
tags: [kmp-compose-visuals]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "An avatar overwritten at the same URL stays old forever in the app, because Coil 3's default cache strategy serves disk hits without revalidation and ignores Cache-Control and ETag."
expected_outcome: "Explains that Coil 3's default strategy never revalidates a disk-cache hit, and adds coil-network-cache-control with CacheControlCacheStrategy through the lambda overload of KtorNetworkFetcherFactory."
---
When a user changes the profile photo in our running app, the backend overwrites the file at the same URL, `https://cdn.example-run.com/avatars/{userId}.jpg`. The CDN serves it with `Cache-Control: max-age=0, must-revalidate` and an `ETag`. Our web app shows the new photo right away.

In the KMP app (Kotlin 2.4.10, CMP 1.11.1, Coil 3.5.0 with coil-network-ktor3, Ktor 3.1.0) the old photo stays, on the uploader's phone and on everyone else's. Force-quit and relaunch does not help. Reinstalling does. Same on Android and iOS.

```kotlin
setSingletonImageLoaderFactory { context ->
    ImageLoader.Builder(context)
        .components { add(KtorNetworkFetcherFactory()) }
        .memoryCache { MemoryCache.Builder().maxSizePercent(context, 0.2).build() }
        .diskCache {
            DiskCache.Builder()
                .directory(imageCacheDir(context))
                .maxSizeBytes(128L * 1024 * 1024)
                .build()
        }
        .build()
}

AsyncImage(
    model = user.avatarUrl,
    contentDescription = stringResource(Res.string.avatar_of, user.name),
    modifier = Modifier.size(40.dp).clip(CircleShape),
)
```

After an upload we already call `imageLoader.memoryCache?.remove(MemoryCache.Key(url))`. The backend team will not change the URL scheme, and the API has no version or `updatedAt` for the photo. Why are the headers ignored, and what is the right client setup?
