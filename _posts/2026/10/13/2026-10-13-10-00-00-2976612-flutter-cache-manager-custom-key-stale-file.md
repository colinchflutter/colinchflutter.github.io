---
layout: post
title: "flutter_cache_manager Custom Keys - Fix Stale Flutter Files and Cache Collisions"
description: "Use flutter_cache_manager custom keys safely: avoid duplicate managers, invalidate the right file, and separate stalePeriod from explicit cache busting."
date: 2026-10-13
tags: [flutter_cache_manager, caching, networking, performance, migration]
comments: true
share: true
---

![Flutter cache manager flow showing URL, cache key, metadata, and file storage](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

The image highlights the boundary to debug: the URL is not always the key used by the cache metadata store.

The production-safe choice is to give each `flutter_cache_manager` configuration one stable manager key, then use a separate explicit cache key for every versioned file. This is useful when a Flutter app caches avatars, downloadable documents, or resized images. It does not replace HTTP cache headers, and it is unnecessary if `Image.network` or `DefaultCacheManager` already meets the requirement.

## The bug is usually two different keys

`flutter_cache_manager` has a manager key and a file key. The manager key identifies the metadata store and cache directory. The file key identifies an item inside that store. Treating them as the same value creates confusing failures:

| Situation | What the app thinks | What actually happens |
| --- | --- | --- |
| Two custom managers use the same manager key | Separate caches | They can touch the same database and files |
| `removeFile(url)` after storing with `key: versionedKey` | The file is deleted | A different key is removed, so the old file remains |
| `stalePeriod` expires | The file disappears immediately | The manager may revalidate or replace it on a later read |
| `emptyCache()` is called after logout | One user’s file is removed | Every file in that manager is removed |

The package documentation recommends using only one manager instance for a given key. It also documents that `removeFile` removes by cache key, not by an arbitrary URL that may have been used to download the file. The cache files are temporary app files, so the operating system may remove them before the configured period as well.

## Use one manager and centralize key construction

The following example keeps the manager key stable while making the file key explicit. The version is part of the file key, so a profile image update does not depend on whether a server sends a useful `Cache-Control` header.

```dart
import 'dart:io';

import 'package:flutter_cache_manager/flutter_cache_manager.dart';

class ProfileImageCache {
  ProfileImageCache._();

  static final instance = ProfileImageCache._();

  static const _managerKey = 'profile-images-v1';
  final CacheManager _manager = CacheManager(
    Config(
      _managerKey,
      stalePeriod: const Duration(days: 7),
      maxNrOfCacheObjects: 200,
    ),
  );

  String keyFor({required String userId, required int revision}) {
    return 'profile/$userId/r$revision';
  }

  Future<File> get({
    required String userId,
    required int revision,
    required String url,
  }) {
    final key = keyFor(userId: userId, revision: revision);
    return _manager.getSingleFile(url, key: key);
  }

  Future<void> remove({required String userId, required int revision}) {
    return _manager.removeFile(
      keyFor(userId: userId, revision: revision),
    );
  }
}
```

The important invariant is that `get` and `remove` call the same `keyFor` method. A URL can be a cache key when no custom key is supplied, but once a custom key is introduced, every later invalidation must use that custom key. Keep user IDs and server revisions normalized; a raw URL containing a changing query string can silently create unbounded entries.

## Choose stale refresh or explicit invalidation

`stalePeriod` is a freshness policy, not a deletion command. When a cached item becomes stale, the manager can check the remote response and return an updated file. With `getFileStream`, the first event can be the cached file and a later event can be the downloaded file. That is good for offline-friendly screens, but it is not the same as “never show the old image.”

Use a revision key when the API already tells you that the content changed:

```dart
final file = await ProfileImageCache.instance.get(
  userId: account.id,
  revision: account.avatarRevision,
  url: account.avatarUrl,
);
```

Use `remove` when the user explicitly replaces or deletes the file. Await it before showing a confirmation or starting a download with the same key. Recent `flutter_cache_manager` releases also changed cache metadata writes to be awaited, so code should not assume that a returned `File` means every metadata operation has already succeeded in older versions.

## Migration checklist for 3.4.x

Before changing a custom cache manager, check these cases rather than only testing a fresh install:

- Keep the manager key unchanged if the existing cache can be reused. Changing it creates a new cache namespace and leaves the old temporary files for normal cleanup.
- If the manager configuration changes, use a new manager key deliberately and document the expected temporary disk increase.
- Do not create two managers with the same key but different `stalePeriod`, repository, or file-system settings.
- Test `getSingleFile` with an old cache entry, `getFileStream` with a stale entry, and `removeFile` with both the URL and the actual custom key.
- Treat a cache miss as normal. Cache files live in temporary storage and can be deleted by the OS.
- Use `emptyCache()` only for a cache-wide reset, such as a deliberate “clear downloads” action.

The package’s current documentation and changelog are the authority for the installed version: [`flutter_cache_manager` package docs](https://pub.dev/packages/flutter_cache_manager), [`CacheManager` API](https://pub.dev/documentation/flutter_cache_manager/latest/flutter_cache_manager/CacheManager-class.html), and [the 3.4.x changelog](https://pub.dev/packages/flutter_cache_manager/changelog). A reported stale-period issue is also a useful reminder that expiry, cleanup scheduling, and explicit deletion are separate behaviors: [issue #446](https://github.com/Baseflow/flutter_cache_manager/issues/446).

For a small app that only needs ordinary HTTP image caching, do not add a custom manager. Use the default manager. Add a custom manager when you need a distinct retention policy, a bounded cache, a custom file service, or a cache key that represents content revision rather than the request URL.

The short rule is: one configuration, one manager key; one resource version, one file key; one invalidation path that uses the exact same key builder.
