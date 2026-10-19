---
layout: post
title: "shared_preferences Isolate Cache - Stop Flutter Background Reads Going Stale"
description: "Fix stale shared_preferences values across Flutter isolates and background engines with reload boundaries, SharedPreferencesAsync, and a safe ownership rule."
date: 2026-10-19
tags: [shared_preferences, caching, Flutter, Android, performance]
comments: true
share: true
---

![Flutter shared_preferences cache and isolate read paths](/assets/images/shared-preferences-async-migration-cache-paths.png)

The key boundary is the separate in-memory cache each isolate keeps before it reaches the native preference store.

The safe choice depends on who can change the preference. A foreground-only settings screen can keep `SharedPreferencesWithCache` for fast synchronous reads. An app that reads or writes the same key from an FCM background handler, another isolate, or another Flutter engine should use `SharedPreferencesAsync`, or explicitly reload the cached instance before reading. This is not a fix for critical data: the package itself does not guarantee that a returned write has reached durable storage.

The picture to watch is not “one app, one singleton”. Each isolate has its own memory, so a cached value in the main isolate is only a snapshot.

## The failure that looks like a random preference bug

Consider a notification handler that records `lastMessageId` while the app is backgrounded. The handler can run in a separate Android isolate. When the user returns to the app, the main isolate may still report the value loaded before the notification arrived.

| Situation | Cached API behaviour | Better boundary |
| --- | --- | --- |
| One isolate owns a settings screen | Fast synchronous getters | `SharedPreferencesWithCache` |
| Background isolate writes a value | Main cache can remain stale | `reload()` before the read |
| Multiple isolates or engines read the same keys | Each cache has a different snapshot | `SharedPreferencesAsync` |
| Token, payment state, or recoverable business data | A preference is too weak | Use a database or server as the source of truth |

The important distinction is between a stale read and a lost write. `reload()` addresses the first. It cannot turn a small key-value store into a transaction log, and it cannot make an immediately terminated process guarantee persistence.

## Reproduce the stale read with two isolates

The following example models the production shape without depending on Firebase. The worker writes a value, while the main isolate keeps an already-created cached instance.

```dart
import 'dart:isolate';

import 'package:shared_preferences/shared_preferences.dart';

Future<void> writeFromWorker(SendPort sendPort) async {
  final prefs = await SharedPreferences.getInstance();
  await prefs.setInt('unread_count', 7);
  sendPort.send(null);
}

Future<void> demo() async {
  final mainPrefs = await SharedPreferences.getInstance();
  await mainPrefs.setInt('unread_count', 0);

  final ready = ReceivePort();
  await Isolate.spawn(writeFromWorker, ready.sendPort);
  await ready.first;

  // This can still be 0: mainPrefs has its own cache.
  final stale = mainPrefs.getInt('unread_count');

  await mainPrefs.reload();
  final fresh = mainPrefs.getInt('unread_count'); // 7

  print('before reload: $stale, after reload: $fresh');
}
```

The exact background entry point varies by plugin. `firebase_messaging` can create an Android background isolate, and a plugin can create another engine. The storage boundary is the same: constructing a second `SharedPreferences` object does not merge the first object’s in-memory map.

## Choose the async API when reload becomes policy

`SharedPreferencesAsync` does not maintain a local cache. A getter is asynchronous because it asks the host platform for the current value. That adds an await to call sites, but it removes the question “which isolate loaded this value, and when?”

```dart
import 'package:shared_preferences/shared_preferences.dart';

class NotificationPreferences {
  NotificationPreferences({SharedPreferencesAsync? store})
      : _store = store ?? SharedPreferencesAsync();

  final SharedPreferencesAsync _store;

  Future<int> unreadCount() async {
    return await _store.getInt('unread_count') ?? 0;
  }

  Future<void> setUnreadCount(int value) {
    return _store.setInt('unread_count', value);
  }
}
```

Use `SharedPreferencesWithCache` when reads dominate and one isolate owns the keys. Its cache options can also restrict the allowed keys, which makes accidental access easier to detect. Use the async API for values that another execution context can update, even if the number of keys is small.

Do not mix old and new objects casually. A migration that changes the API is only half complete if an authentication plugin or a background handler continues writing through the legacy API. Map every writer before changing the reader.

## A migration boundary that can be reviewed

For an existing app, keep the legacy instance only long enough to copy its values, then make one API the owner of each key. The package supplies a migration helper for the legacy store.

```dart
import 'package:shared_preferences/shared_preferences.dart';
import 'package:shared_preferences/util/legacy_to_async_migration_util.dart';

Future<SharedPreferencesAsync> createPreferences() async {
  final legacy = await SharedPreferences.getInstance();

  await migrateLegacySharedPreferencesToSharedPreferencesAsyncIfNecessary(
    legacySharedPreferencesInstance: legacy,
    sharedPreferencesAsyncOptions: const SharedPreferencesOptions(),
    migrationCompletedKey: 'preferences_async_migration_v1',
  );

  return SharedPreferencesAsync();
}
```

Keep `migrationCompletedKey` stable. Changing or deleting it makes the helper reconsider the migration on a later launch. Also check the Android backend: the async API defaults to DataStore, while an app that must read an existing native `SharedPreferences` file may need `SharedPreferencesAsyncAndroidOptions` with the Android SharedPreferences backend.

## Production checklist

- List every writer: UI isolate, `Isolate.run`, FCM background handler, native code, and plugins with a background engine.
- Mark keys that are only UI hints. Those can tolerate a stale cache; auth or sync decisions usually cannot.
- If the cached API remains, call `reload()` at a deliberate boundary such as app resume or after a background job completes.
- If most reads need `reload()`, replace that cache with `SharedPreferencesAsync` instead of spreading reload calls through widgets.
- Exercise a cold start, background notification, foreground return, and process restart. A hot reload is not evidence that the native store and every isolate agree.
- Keep critical records outside `shared_preferences`; store a durable state machine in a database or on the server.

The practical rule is simple: cache ownership must be explicit. `SharedPreferencesWithCache` is a useful local read model, `reload()` is a synchronization point, and `SharedPreferencesAsync` is the less surprising default when more than one execution context can touch the same key.

Sources: [shared_preferences README](https://github.com/flutter/packages/blob/main/packages/shared_preferences/shared_preferences/README.md), [Flutter issue on multiple isolates and reload](https://github.com/flutter/flutter/issues/101691), and [Flutter isolate documentation](https://docs.flutter.dev/perf/isolates).
