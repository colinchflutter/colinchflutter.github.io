---
layout: post
title: "shared_preferences Migration - Move Flutter Apps to SharedPreferencesAsync Without Stale Cache"
description: "Migrate Flutter shared_preferences safely from the legacy cached API to SharedPreferencesAsync, with a decision table, startup code, and isolate cache traps."
date: 2026-11-16
tags: [shared_preferences, Flutter, migration, caching, Android, iOS]
comments: true
share: true
---

![shared_preferences migration from a cached Flutter store to an async platform store](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=1200&q=80)

If a production Flutter app still uses the legacy `SharedPreferences` API, migrate its stored values before replacing every read with `SharedPreferencesAsync`. The async API is the safer choice when a background engine, another isolate, native code, or another app component can update the same key. It is not a drop-in performance upgrade: every read is asynchronous, and values used during the first frame need an explicit initialization step.

## The failure is usually a cache mismatch

`SharedPreferences` loads a local cache during initialization. `SharedPreferencesWithCache` has a similar model, while `SharedPreferencesAsync` reads the platform store without a Dart-side cache. This distinction matters when the foreground engine and a background context both touch a preference.

| Situation | Better choice | Reason | Trade-off |
| --- | --- | --- | --- |
| One Flutter engine owns small UI settings | `SharedPreferencesWithCache` | Synchronous getters after setup | Must call `reload()` when another writer may exist |
| Firebase Messaging or another engine can write | `SharedPreferencesAsync` | Reads the latest platform value | Every getter returns a `Future` |
| Existing released app using legacy `SharedPreferences` | Run migration, then choose one API | Preserves users' old values | Startup becomes an explicit async gate |
| Access token, refresh token, or secret | Neither API | The plugin is not a critical secret store | Use a secure storage design instead |

The official package documentation calls the old `SharedPreferences` API legacy and recommends `SharedPreferencesAsync` or `SharedPreferencesWithCache` for new code. It also warns that each isolate and engine has its own cached singleton. A `false` value in one cache does not prove that the platform store still contains `false`.

## Migrate once, before reads and writes

The package provides `migrateLegacySharedPreferencesToSharedPreferencesAsyncIfNecessary`. Keep the legacy instance configured exactly as the app used it, including a custom prefix. Store the completion marker in the target store, and use a key name that will never be used by feature code.

This startup service makes migration a prerequisite for constructing the rest of the preference layer:

```dart
import 'package:shared_preferences/shared_preferences.dart';
import 'package:shared_preferences/util/legacy_to_async_migration_util.dart';

class PreferenceBootstrap {
  static const _migrationKey = '__prefs_migrated_to_async_v1';

  static Future<SharedPreferencesAsync> initialize() async {
    final legacy = await SharedPreferences.getInstance();

    const options = SharedPreferencesOptions();
    await migrateLegacySharedPreferencesToSharedPreferencesAsyncIfNecessary(
      legacySharedPreferencesInstance: legacy,
      sharedPreferencesAsyncOptions: options,
      migrationCompletedKey: _migrationKey,
    );

    return SharedPreferencesAsync(options: options);
  }
}
```

Call it before `runApp` or expose its `Future` through the dependency graph. Do not start a second `SharedPreferences.getInstance()` path in a repository while this migration is running. That creates a race where old code can write a value after the copy and before the new API becomes the sole writer.

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final preferences = await PreferenceBootstrap.initialize();
  runApp(App(preferences: preferences));
}
```

The migration helper is safe to call on every launch as long as the completion key is stable and remains in the target store. That does not make a bad configuration safe. Changing the prefix, reusing the marker for a feature flag, or creating the new Android backend with different options can make old values appear to disappear.

## Make the cache decision explicit

If synchronous reads are important for a settings-heavy screen and only the foreground engine writes, `SharedPreferencesWithCache` is reasonable. Limit its keys at construction time so unrelated native values cannot break initialization, and call `reload()` at a known boundary after returning from a background flow.

```dart
final prefs = await SharedPreferencesWithCache.create(
  cacheOptions: const SharedPreferencesWithCacheOptions(
    allowList: <String>{'theme_mode', 'onboarding_complete'},
  ),
);

final theme = prefs.getString('theme_mode') ?? 'system';
```

Use `SharedPreferencesAsync` for values whose freshness matters more than synchronous access. On Android, its default backend is DataStore Preferences, but the package also supports the older Android SharedPreferences backend when an existing native integration requires it. Matching the backend is part of migration; changing it casually is not a harmless refactor.

## A small release checklist

- [ ] Identify every writer: foreground isolate, background engine, native code, and tests.
- [ ] Preserve the old prefix and use a unique completion key.
- [ ] Run the migration before the first preference read or write.
- [ ] Pick one API per preference repository; do not mix old and new writers indefinitely.
- [ ] Test an upgrade install with existing values, not only a fresh install.
- [ ] Test Android DataStore versus legacy SharedPreferences when native code owns the file.
- [ ] Treat tokens and other secrets as outside this plugin's security boundary.

The practical fix is not “replace the class name everywhere.” It is to define who owns each key, copy old data once, then choose cached or uncached reads according to the number of writers. That turns a stale-cache report into a reproducible migration policy.

References checked 2026-11-16: [shared_preferences package documentation](https://pub.dev/packages/shared_preferences), [migration utility API](https://pub.dev/documentation/shared_preferences/latest/util_legacy_to_async_migration_util/migrateLegacySharedPreferencesToSharedPreferencesAsyncIfNecessary.html), and [Flutter package source migration notes](https://github.com/flutter/packages/blob/main/packages/shared_preferences/shared_preferences/README.md).
