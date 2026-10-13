---
layout: post
title: "shared_preferences Windows Migration - Stop Async and Legacy Stores Overwriting Each Other"
description: "A Windows-specific shared_preferences migration trap: why the legacy and async APIs can erase each other's keys, and how to stage a safe Flutter rollout."
date: 2026-10-14
tags: [shared_preferences, migration, Windows, persistence, Flutter]
comments: true
share: true
---

![Flutter shared_preferences migration boundary between legacy and async stores](/assets/images/shared-preferences-async-migration-cache-paths.png)

If your Flutter app ships on Windows, keep `SharedPreferences` and `SharedPreferencesAsync` out of the same live write path during migration. On the current Windows implementation, each store can load its own in-memory map and later replace the shared `shared_preferences.json` file; the last writer can silently remove keys written by the other store. Android and iOS teams may safely follow the package migration helper, but a Windows app with legacy plugins needs a staged decision instead.

## Why the normal migration recipe is risky on Windows

The package recommends `migrateLegacySharedPreferencesToSharedPreferencesAsyncIfNecessary`. That helper is useful because it copies existing values and records a completion key. The problem is the overlap: migration intentionally creates a legacy instance and an async instance in one process. A legacy plugin that writes after the copy can replace the file with its older snapshot. The async store can do the same in the opposite direction.

| Situation | Safer choice | Reason |
| --- | --- | --- |
| Android/iOS, one API owns all writes | `SharedPreferencesAsync` | No local cache; reads go to the host store |
| One isolate owns settings and sync getters matter | `SharedPreferencesWithCache` | Fast reads, with explicit reload boundaries |
| Windows still has plugins using legacy `SharedPreferences` | Keep legacy temporarily | Avoid two writers with independent snapshots |
| Windows after every consumer has migrated | Switch once to the new API | No legacy writer remains to erase new keys |

## Make the platform decision before constructing preferences

Do not create both implementations globally and choose one later. Put the boundary in bootstrap code, then inject one repository into the rest of the app. This example deliberately keeps Windows on the legacy API until its dependency graph is clear:

```dart
import 'dart:io' show Platform;
import 'package:shared_preferences/shared_preferences.dart';
import 'package:shared_preferences/util/legacy_to_async_migration_util.dart';

Future<Object> createPreferences() async {
  if (Platform.isWindows) {
    // Keep one writer while legacy plugins remain in the graph.
    return SharedPreferences.getInstance();
  }

  final legacy = await SharedPreferences.getInstance();
  await migrateLegacySharedPreferencesToSharedPreferencesAsyncIfNecessary(
    legacySharedPreferencesInstance: legacy,
    sharedPreferencesAsyncOptions: const SharedPreferencesOptions(),
    migrationCompletedKey: 'prefs_migration_v1_done',
  );
  return SharedPreferencesAsync();
}
```

The `Object` return type is intentional only for a small bootstrap example. In production, expose a repository interface so feature code never calls `getInstance()` on its own. The migration key must be stable; changing or deleting it can repeat the copy. Also keep the old key prefix and backend options unchanged until you have verified existing installations, because a prefix change makes old values look missing rather than migrated.

## Windows rollout checklist

- Search packages and plugins for `SharedPreferences.getInstance()` before enabling the new store.
- On Windows, choose one writer per release. Do not run a background callback that still uses the legacy API beside the new repository.
- Back up a test profile's `shared_preferences.json`, write a legacy-only key and a new-repository key, then relaunch and verify both survive.
- Roll out the new API only after the last legacy writer is removed or upgraded.
- Treat missing settings as a migration failure, not as a default-value case; log the platform and migration phase without logging secrets.

The package documentation currently recommends the async API for multi-isolate or multi-engine data, while the open Flutter issue documents the Windows overwrite condition. That means the correct fix is platform-specific orchestration, not blindly replacing every `SharedPreferences` type. If the app does not target Windows, this particular collision does not justify delaying the normal migration.

Sources: [shared_preferences README](https://github.com/flutter/packages/blob/main/packages/shared_preferences/shared_preferences/README.md), [migration API reference](https://pub.dev/documentation/shared_preferences/latest/util_legacy_to_async_migration_util/), and [Flutter issue #192759](https://github.com/flutter/flutter/issues/192759).
