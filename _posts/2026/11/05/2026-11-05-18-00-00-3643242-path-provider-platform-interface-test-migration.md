---
layout: post
title: "path_provider PlatformInterface - Migrate Flutter Tests Without Mocking the Wrong Channel"
description: "Fix path_provider tests that fail after the platform-interface migration by replacing brittle MethodChannel mocks with a typed PathProviderPlatform fake and a small storage wrapper."
date: 2026-11-05
tags: [path_provider, testing, migration, Android, iOS]
comments: true
share: true
---

![Flutter path_provider platform interface test migration](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?auto=format&fit=crop&w=1600&q=80)

The safe migration for a Flutter app that tests `path_provider` is to stop mocking the plugin's private `MethodChannel` and register a typed `PathProviderPlatform` fake instead. This applies when a unit or widget test calls `getApplicationDocumentsDirectory()`, `getTemporaryDirectory()`, or another top-level API. It does not mean the fake should replace an integration test that needs a real Android or iOS filesystem.

The important boundary is visible in the current `path_provider` 2.1.6 documentation: the plugin uses a platform interface, and its testing guidance explicitly says to mock `PathProviderPlatform` rather than `PlatformChannel`. The public function still returns a `Directory`; only the seam underneath it changed.

## Why the old test breaks

A common older test installs a handler on `plugins.flutter.io/path_provider` and asserts the method name or channel arguments. That test is coupled to one native implementation. It can fail in three different ways after an upgrade:

| Symptom | Hidden assumption | Better seam |
| --- | --- | --- |
| `MissingPluginException` | A Dart test has a host implementation | `PathProviderPlatform.instance` |
| Handler never runs | The plugin still uses the mocked channel | The federated platform interface |
| Test passes on one host only | Every platform uses the same channel implementation | A fake that returns explicit paths |

The failure is not proof that the application lost filesystem access. It often means the test is intercepting an implementation detail that the package no longer promises.

This is the behavior to keep under test:

```dart
import 'dart:io';

import 'package:path_provider/path_provider.dart';

class DraftStore {
  Future<File> fileFor(String id) async {
    final directory = await getApplicationSupportDirectory();
    return File('${directory.path}/draft-$id.json');
  }
}
```

`DraftStore` should not know whether the path came from Android's data directory or iOS's support directory. The test only needs a deterministic directory.

## Replace the channel mock with a platform fake

Add the interface package as a test-visible dependency if your project imports it directly. The current package documentation lists `path_provider_platform_interface` as the package that owns the typed interface.

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  path_provider_platform_interface: ^2.1.3
```

Extend the interface rather than implementing it. The API documentation recommends `extends` because new methods can receive a default implementation; `implements` makes every added method an immediate compile failure.

```dart
import 'package:path_provider_platform_interface/path_provider_platform_interface.dart';

class FakePathProvider extends PathProviderPlatform {
  FakePathProvider({this.supportPath = '/test/app-support'});

  final String supportPath;

  @override
  Future<String?> getApplicationSupportPath() async => supportPath;
}
```

Register the fake for one test and restore the original instance in `tearDown`. Restoring matters because `PathProviderPlatform.instance` is global state; without it, a later test can silently use the previous test's directory.

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:path_provider/path_provider.dart';
import 'package:path_provider_platform_interface/path_provider_platform_interface.dart';

void main() {
  late PathProviderPlatform original;

  setUp(() {
    original = PathProviderPlatform.instance;
    PathProviderPlatform.instance =
        FakePathProvider(supportPath: '/test/app-support');
  });

  tearDown(() {
    PathProviderPlatform.instance = original;
  });

  test('puts drafts under application support', () async {
    final file = await DraftStore().fileFor('42');

    expect(file.path, '/test/app-support/draft-42.json');
  });
}
```

The assertion checks the app's decision, not a platform channel string. A channel rename, a federated implementation change, or a different host OS does not change what this test is responsible for.

## Do not use the wrong directory just to make the test green

`path_provider` exposes several locations with different contracts. The current API documentation describes the documents directory as suitable for user-generated or non-recreatable data, while cache data can be cleared and support data is for application support files. Pick the fake method that matches the production method.

| Production call | Test fake method | Suitable example |
| --- | --- | --- |
| `getTemporaryDirectory()` | `getTemporaryPath()` | resized image or upload scratch file |
| `getApplicationCacheDirectory()` | `getApplicationCachePath()` | rebuildable HTTP cache |
| `getApplicationSupportDirectory()` | `getApplicationSupportPath()` | private settings or draft metadata |
| `getApplicationDocumentsDirectory()` | `getApplicationDocumentsPath()` | user-created export |

If the code uses `getApplicationDocumentsDirectory()` but the test fake only overrides support, the inherited default may return `null` or throw. That is a useful failure: it tells you that the test double does not describe the production contract.

For a service that needs more than one location, make the fake explicit:

```dart
class FakePathProvider extends PathProviderPlatform {
  FakePathProvider({
    this.documentsPath = '/test/documents',
    this.cachePath = '/test/cache',
  });

  final String documentsPath;
  final String cachePath;

  @override
  Future<String?> getApplicationDocumentsPath() async => documentsPath;

  @override
  Future<String?> getApplicationCachePath() async => cachePath;
}
```

## When a wrapper is better than mocking the plugin

Flutter's testing guidance recommends wrapping plugin calls in an application-owned API when possible. That gives the project a stable seam even if the plugin interface changes again.

```dart
abstract interface class AppDirectories {
  Future<Directory> support();
}

class PluginAppDirectories implements AppDirectories {
  @override
  Future<Directory> support() => getApplicationSupportDirectory();
}
```

Most business-logic tests can now fake `AppDirectories` without importing `path_provider_platform_interface`. Use the platform fake when you are specifically testing the adapter; use the wrapper fake everywhere else.

## Migration checklist

- Remove `TestDefaultBinaryMessenger` handlers for `path_provider` from app-level unit and widget tests.
- Search for direct `MethodChannel('plugins.flutter.io/path_provider')` usage.
- Override the exact `PathProviderPlatform` method used by production code.
- Register the fake through `PathProviderPlatform.instance`.
- Restore the original instance in `tearDown`.
- Keep at least one integration test for real platform registration and filesystem behavior.
- Run the test suite on a clean process when diagnosing order-dependent failures.

The short rule is: mock the package-owned interface for adapter tests, mock your own directory abstraction for application tests, and reserve platform channels for cases where no higher-level seam exists. That keeps a package upgrade from turning a path decision into a channel-maintenance task.

Sources checked 2026-11-05: [path_provider package documentation](https://pub.dev/packages/path_provider), [PathProviderPlatform API](https://pub.dev/documentation/path_provider_platform_interface/latest/path_provider_platform_interface/PathProviderPlatform-class.html), [Flutter plugin testing guidance](https://github.com/flutter/website/blob/main/sites/docs/src/content/testing/plugins-in-tests.md), and [getApplicationDocumentsDirectory API](https://pub.dev/documentation/path_provider/latest/path_provider/getApplicationDocumentsDirectory.html).
