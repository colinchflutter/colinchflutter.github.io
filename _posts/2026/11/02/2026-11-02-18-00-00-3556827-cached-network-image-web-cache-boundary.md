---
layout: post
title: "cached_network_image Web Cache - Keep Flutter Image Caching Predictable Across Platforms"
description: "cached_network_image caches files on mobile but has limited Web support. This guide separates memory, disk, and browser caching and gives a safe cacheKey and fallback policy."
date: 2026-11-02
tags: [cached_network_image, caching, networking, performance, Web, Flutter]
comments: true
share: true
---

![Flutter cached_network_image cache boundary between mobile disk storage and Web browser caching](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

If a Flutter app must show an avatar or catalog image offline, `cached_network_image` is a good fit on Android and iOS when its disk cache is available. It is not a cross-platform guarantee that the same file cache exists: the package documentation describes Web support as limited and says it currently does not include caching. A shared Flutter widget therefore needs a platform-aware freshness and offline policy. This does not matter for a disposable placeholder or an image that is always fetched from a CDN.

## The failure is usually an incorrect cache promise

The widget API looks identical on mobile and Web, so it is easy to read `CachedNetworkImage(imageUrl: ...)` as “download once and reuse everywhere.” In practice, three different caches are involved:

| Layer | Android/iOS | Flutter Web | What the app can promise |
|---|---|---|---|
| Flutter in-memory `ImageCache` | Yes | Yes | Avoids repeated decode while the key is retained |
| `flutter_cache_manager` file cache | Normal path | Not provided by the original package | Offline reuse after a restart is platform-dependent |
| Browser/CDN HTTP cache | May exist below the client | Controlled by browser headers | Freshness follows URL and response headers |

The practical bug appears after an API changes an image at the same URL. A mobile user can see an old disk file until its cache entry expires, while a Web user may request the image again and receive a different result from the browser or CDN. The reverse is also possible: a long-lived browser cache can keep a stale asset after the server changed it.

## Keep the URL and cache key versioned together

For user-generated images, a stable URL plus an explicit revision is safer than clearing every cache after an update. The revision can be an object version, an ETag-derived token, or a server-provided `updatedAt` value. Do not use a user name as the key when the image can change or when two accounts can share a name.

Put the cache-key policy in one adapter so every screen uses the same value. The code below is deliberately independent of a state-management package.

```dart
import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter/material.dart';

class VersionedImage extends StatelessWidget {
  const VersionedImage({
    required this.url,
    required this.revision,
    super.key,
  });

  final String url;
  final String revision;

  @override
  Widget build(BuildContext context) {
    final cacheKey = '$url#$revision';

    return CachedNetworkImage(
      imageUrl: url,
      cacheKey: cacheKey,
      fit: BoxFit.cover,
      placeholder: (context, url) => const ColoredBox(
        color: Color(0xFFE9EDF2),
        child: Center(child: CircularProgressIndicator()),
      ),
      errorWidget: (context, url, error) => const Icon(Icons.broken_image),
    );
  }
}
```

This separates two concerns: the URL is where bytes come from, and `cacheKey` identifies which version is safe to reuse. A query parameter such as `?v=42` can also work, but only when the CDN and origin treat it as part of the resource identity. If the backend signs URLs, prefer the server’s canonical versioned URL rather than inventing a second signature.

## Do not treat a stale file as an offline guarantee

On mobile, `CachedNetworkImage` delegates storage to `flutter_cache_manager`. A cached entry can be expired, evicted, deleted by the operating system, or unavailable because its metadata write failed. The current `flutter_cache_manager` changelog also records fixes around awaiting metadata persistence and treating failed metadata reads as cache misses. Code should therefore handle the error state even when a file “was cached yesterday.”

For a product image, a stale image is often better than a blank cell. For a security-sensitive document or a price label, stale data may be misleading. Make that decision explicit:

| Content | Offline fallback | Cache policy |
|---|---|---|
| Avatar or article thumbnail | Show last known image | Versioned key, stale display allowed |
| Product price artwork | Show placeholder plus “refresh required” | Short freshness window, no silent stale claim |
| Signed document | Show an error and retry action | Never present an expired file as current |

When the server returns a new revision, let the old key become unreachable and allow normal cleanup to remove it. Calling `emptyCache()` on every login or deploy creates unnecessary network traffic and can make a poor connection look like a content outage. Explicitly remove one known key only when a user deletes or replaces that asset.

## Web needs a different acceptance test

The package page says the Web implementation has minimal support and currently does not include caching. That means a Web test should verify rendering, CORS, browser cache headers, and the behavior after a reload—not the presence of a local cache file. `flutter_cache_manager` is also not a universal Web storage abstraction just because it is used by the mobile implementation.

Use a small matrix before claiming “offline image support”:

```text
Android/iOS: first load -> kill app -> reopen offline -> old revision is acceptable
Web:         first load -> reload -> inspect request/cache headers -> new revision appears
All:         404, expired URL, CORS failure, and corrupted bytes show an error state
```

For Web, configure the origin or CDN deliberately. Long-lived immutable assets should have a content-hashed or revisioned URL. Mutable URLs need revalidation headers, otherwise the browser is allowed to reuse an old response. Flutter Web’s own deployment guidance makes the same distinction for hashed entrypoints and mutable bootstrap files: cache identity and revalidation belong to the server contract, not only to a widget.

## Reproducible checklist

- [ ] Decide whether stale content is acceptable for this image type.
- [ ] Pass a revision-aware `cacheKey`; do not key mutable content by a display name.
- [ ] Test a clean install and a restart with the network disabled on Android/iOS.
- [ ] Test Web with a hard reload and inspect response `Cache-Control`, `ETag`, and CORS headers.
- [ ] Replace one image at the same logical record and verify the new revision is rendered.
- [ ] Test 404, timeout, revoked signed URL, and cache metadata failure paths.
- [ ] Document that Web support does not imply a persistent file cache.

The safe boundary is simple: use `cached_network_image` for a consistent widget and cache-key policy, but keep persistence and freshness as platform-specific product decisions. Mobile can usually reuse a file through `flutter_cache_manager`; Web should rely on versioned URLs and correct HTTP headers. A green widget test only proves that an image can render—it does not prove offline recovery or cache invalidation.

Sources: [cached_network_image package documentation](https://pub.dev/packages/cached_network_image), [flutter_cache_manager changelog](https://pub.dev/packages/flutter_cache_manager/changelog), and [Flutter Web cache guidance](https://docs.flutter.dev/platform-integration/web/faq) (checked 2026-11-02).
