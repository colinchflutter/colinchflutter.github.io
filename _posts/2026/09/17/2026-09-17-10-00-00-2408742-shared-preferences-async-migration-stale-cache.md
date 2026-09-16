---
layout: post
title: "shared_preferences Migration - Fix Stale Flutter Cache Across Isolates"
description: "Migrate Flutter shared_preferences safely to SharedPreferencesAsync or WithCache, preserve existing keys, and prevent stale values across isolates and background engines."
date: 2026-09-17
tags: [shared_preferences, caching, migration, performance, Android, iOS]
comments: true
share: true
---

![Flutter shared_preferences migration from a cached API to platform-direct storage](/assets/images/shared-preferences-async-migration-cache-paths.png)

If a Flutter app reads preferences from a background isolate, a notification engine, or native code, `SharedPreferencesAsync` is usually the safer migration target. If the app only reads its own small settings from one isolate and synchronous getters matter, `SharedPreferencesWithCache` is a better fit. The legacy `SharedPreferences` API can keep working during a staged rollout, but mixing it with a new API without a migration boundary can expose stale values or overwrite the same store unexpectedly.

The choice is not simply “old API versus new API.” It is a decision about who owns the latest value and when a read is allowed to use memory.

## The production failure is a cache ownership problem

`SharedPreferences` and `SharedPreferencesWithCache` keep an in-memory cache. That makes repeated reads convenient, but each isolate or engine can have a different snapshot. A foreground Flutter engine and a background context created by a plugin can therefore disagree about a flag such as `syncEnabled`, `lastSyncAt`, or `accessToken`.

The official package documentation calls out three situations where this matters:

| Environment | What can go stale | Safer default |
| --- | --- | --- |
| One isolate, app-owned settings | Rarely stale after startup | `WithCache` |
| Multiple isolates or engine instances | Cached reads from another context | `Async` |
| Native code changes preferences | Dart cache does not know the change | `Async` or explicit reload |

The `shared_preferences` changelog introduced `SharedPreferencesAsync` and `SharedPreferencesWithCache` in 2.3.0, while the migration helper arrived in 2.4.0. This is a package API transition, not a reason to delete the old keys and start over.

## Pick the target API before changing call sites

Use this decision table for a real application rather than changing every `getBool` mechanically.

| Requirement | API | Trade-off |
| --- | --- | --- |
| Latest platform value on every read | `SharedPreferencesAsync` | Every getter is asynchronous; more platform calls |
| Fast synchronous getters after initialization | `SharedPreferencesWithCache` | Must reload when another context may write |
| Critical data such as credentials or durable records | Neither | Preference storage is not a database or secret vault |

`SharedPreferencesAsync` does not use a local cache. Its reads go through the host platform, so it is a strong boundary for data shared with another engine. The default Android backend is DataStore Preferences; choose the Android SharedPreferences backend only when the app must interoperate with values written by code that already uses that backend.

## Keep the one-time migration separate from the repository

Run the official migration utility during startup before the new repository is used. Keep the completion key stable. The utility can be called on every launch; it will not repeat the migration unless its completion marker is removed.

The following bootstrap keeps the legacy instance alive only for the migration step:

```dart
import 'package:shared_preferences/shared_preferences.dart';
import 'package:shared_preferences/util/legacy_to_async_migration_util.dart';

Future<SharedPreferencesAsync> createPreferences() async {
  final legacy = await SharedPreferences.getInstance();

  await migrateLegacySharedPreferencesToSharedPreferencesAsyncIfNecessary(
    legacySharedPreferencesInstance: legacy,
    sharedPreferencesAsyncOptions: const SharedPreferencesOptions(),
    migrationCompletedKey: 'prefs_migration_v1_completed',
  );

  return SharedPreferencesAsync();
}
```

The important detail is the stable marker, not the name of the Dart variable. Do not use a temporary key, a version that changes on every release, or a key that a “clear settings” button can remove. If the app intentionally supports rollback to a build that only understands the legacy API, define that rollback policy before shipping the migration. A forward-only migration can make a downgrade harder to reason about.

After bootstrap, inject the returned instance into one repository and stop creating `SharedPreferences.getInstance()` in feature code:

```dart
class SettingsStore {
  SettingsStore(this._prefs);

  final SharedPreferencesAsync _prefs;

  Future<bool> isSyncEnabled() async {
    return await _prefs.getBool('sync_enabled') ?? true;
  }

  Future<void> setSyncEnabled(bool value) {
    return _prefs.setBool('sync_enabled', value);
  }
}
```

This repository boundary prevents one screen from reading a cached legacy object while another service writes through the async object. It also makes the asynchronous change visible in method signatures, so callers cannot accidentally assume that a preference read is free or synchronous.

## When `WithCache` is the right answer

An app that owns all writes from one isolate may not need platform-direct reads. Create a cache with an allowlist instead of giving every preference key unrestricted access:

```dart
final prefs = await SharedPreferencesWithCache.create(
  cacheOptions: const SharedPreferencesWithCacheOptions(
    allowList: <String>{'theme_mode', 'sync_enabled'},
  ),
);

final theme = prefs.getString('theme_mode') ?? 'system';
```

The allowlist is a useful failure boundary: a misspelled key fails near the repository instead of silently becoming a new setting. If native code or another engine can change those keys, call `reloadCache()` before reading, or use `SharedPreferencesAsync` for that repository. Reloading every time is a signal that the cache is the wrong abstraction.

## Prefixes can make a successful migration look empty

The legacy API normally reads keys with the `flutter.` prefix. Prefix handling is internal, so application code should keep using `sync_enabled`, not manually prepend `flutter.` to ordinary calls. If migrating from a native implementation, changing the prefix changes which keys are visible. Existing values do not automatically become visible under a new prefix; they need an explicit transformation or a compatibility read.

Be careful when removing the prefix entirely. The plugin may encounter unrelated native values with unsupported types while initializing. The official documentation recommends an allowlist for that case. Test an upgrade from an installed legacy build, not only a clean install, because a clean install cannot reveal a key-visibility regression.

## Upgrade checklist

- Decide whether another isolate, engine, or native component can write the key.
- Choose `Async` for latest-value semantics; choose `WithCache` only with an explicit reload policy.
- Run the migration helper once per install using a stable completion key.
- Keep preference access behind one repository and remove feature-level legacy instances.
- Verify the Android backend if native Android code reads the same store.
- Test upgrade, force-stop, background callback, logout/login, and clean install separately.
- Do not store secrets or critical transactional data in this package; writes are not guaranteed to be persisted before the method returns.

The practical fix is to make cache ownership explicit. Use `SharedPreferencesAsync` when the preference is shared across execution contexts, `SharedPreferencesWithCache` when a single isolate owns the data and needs fast reads, and the migration helper to preserve existing users. That combination changes the API without turning an app update into a settings reset.

Sources: [shared_preferences README](https://github.com/flutter/packages/blob/main/packages/shared_preferences/shared_preferences/README.md), [shared_preferences changelog](https://github.com/flutter/packages/blob/main/packages/shared_preferences/shared_preferences/CHANGELOG.md), and [a real migration issue in supabase-flutter](https://github.com/supabase/supabase-flutter/issues/1276).
