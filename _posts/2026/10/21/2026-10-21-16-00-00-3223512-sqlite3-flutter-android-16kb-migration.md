---
layout: post
title: "sqlite3 Flutter Migration - Replace sqlite3_flutter_libs Before Android 16 KB Failures"
description: "A production migration guide for sqlite3 2.x to 3.x in Flutter, including transitive native libraries, Android 16 KB checks, and the cases where you should delay the upgrade."
date: 2026-10-21
tags: [sqlite3, Android, migration, performance, testing]
comments: true
share: true
---

![Android 16 KB page-size verification for Flutter native libraries](https://developer.android.com/static/images/guide/practices/play-console-16kb.png)

*The useful signal is the native `.so` file in the release artifact, not the package name shown in `pubspec.yaml`.*

If a Flutter Android release reports `libsqlite3.so` as incompatible with 16 KB page sizes, upgrading Gradle alone is not the fix. Apps using `sqlite3_flutter_libs` should plan the `sqlite3` 3.x migration; apps receiving SQLite through a package such as Amplify should upgrade that parent package first. Stay on 2.x temporarily when your Flutter channel still has unresolved build-hook issues or when your encryption setup has not been migrated.

## Why this failure is easy to misdiagnose

Android 15 introduced 16 KB page-size devices, and Google Play requires support for updates targeting Android 15 (API 35) and higher from February 1, 2027. Android’s guidance says every prebuilt native library must also be rebuilt or replaced. Changing `compileSdk`, `targetSdk`, or the NDK used by the app does not rewrite a `.so` file already shipped inside a dependency.

That is why this dependency graph matters:

| What you see | What it means | Correct action |
| --- | --- | --- |
| `sqlite3_flutter_libs` in your pubspec | Old platform-specific loader | Migrate to `sqlite3` 3.x |
| `libsqlite3.so` but no direct SQLite dependency | A transitive plugin owns it | Upgrade that plugin; do not add a second SQLite package |
| Another vendor `.so` remains unaligned | The package ships a prebuilt binary | Replace or wait for that package release |
| No `.so` files in the APK | No native alignment issue from SQLite | Test the release anyway if the app targets API 35+ |

## The sqlite3 2.x to 3.x change

The upstream migration guide removes `sqlite3_flutter_libs` and `sqlcipher_flutter_libs`. Version 3 uses Dart build hooks to bundle native SQLite assets. It also removes the old `open.overrideFor` customization path and the Android workaround named `applyWorkaroundToOpenSqlite3OnOldAndroidVersions`.

Start with the dependency change:

```yaml
dependencies:
  sqlite3: ^3.0.0
  # Remove sqlite3_flutter_libs
```

Then remove old loader initialization. Keep your database schema and SQL code unchanged, but rebuild from a clean state so an old native artifact is not mistaken for the new one:

```bash
flutter pub get
flutter clean
rm -rf android/.gradle build
flutter build appbundle --release
```

If the app uses SQLCipher, do not blindly replace it with plain SQLite. The 3.x configuration uses a hook user define instead:

```yaml
hooks:
  user_defines:
    sqlite3:
      source: sqlcipher
```

Treat that as a separate compatibility change: open an existing encrypted database in a staging build before shipping.

## Trace the package that owns the binary

When the dependency is indirect, inspect the graph before editing Android files:

```bash
flutter pub deps --style=compact | rg 'sqlite3|amplify_db_common|drift'
unzip -l build/app/outputs/bundle/release/app-release.aab | rg 'libsqlite3|\.so'
```

An Amplify issue demonstrates the trap: `amplify_db_common` pulled an older `eu.simonbinder:sqlite3-native-library`, so the app had `libsqlite3.so` even without a direct SQLite dependency. The durable fix is an Amplify release that updates that native dependency. A Gradle resolution override may make one build pass, but it can break the plugin’s ABI or SQLite expectations, so use it only with a reproduced integration test and a documented rollback.

## Verify the artifact, not just the build log

Use Android Studio APK Analyzer or the Android-provided ELF checker. For a 16 KB test environment, confirm the device reports `16384`, then check zip alignment:

```bash
adb shell getconf PAGE_SIZE
zipalign -c -P 16 -v 4 app-release.apk
```

The Android documentation also recommends testing database open, migration, WAL/checkpoint, encryption, and background-isolate access on a 16 KB emulator. A green Gradle build proves none of those runtime paths.

The practical rule is simple: migrate the package that produces the native library, verify the final AAB, and keep the old dependency only when the build-hook or encryption constraints are explicit and tested.

Sources: [sqlite3 3.x upgrade notes](https://github.com/simolus3/sqlite3.dart/blob/main/UPGRADING_TO_V3.md), [Android 16 KB page-size guidance](https://developer.android.com/guide/practices/page-sizes), [Amplify transitive SQLite issue](https://github.com/aws-amplify/amplify-flutter/issues/6176).
