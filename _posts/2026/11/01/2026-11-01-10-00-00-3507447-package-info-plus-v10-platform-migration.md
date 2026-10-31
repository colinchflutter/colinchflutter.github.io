---
layout: post
title: "package_info_plus 10 Migration - Fix Flutter Platform Builds and Stale App Versions"
description: "Migrate package_info_plus 10 in Flutter without confusing SDK constraints, stale iOS metadata, web version.json failures, or unreliable installer-store values."
date: 2026-11-01
tags: [package_info_plus, migration, Android, iOS, Web, testing]
comments: true
share: true
---

![package_info_plus platform metadata migration in Flutter](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

`package_info_plus` 10 is a good upgrade for an app that needs a single API for version, build number, or installer metadata across platforms. It is not a drop-in dependency bump for an older Flutter project: the current package page requires Flutter 3.38.1, Dart 3.10, Java 17, Android Gradle Plugin 8.12.1, and Gradle 8.13. If the app still has an older toolchain, keep the previous compatible package or upgrade the toolchain as a separate change. Mixing both changes in one emergency release makes the failure hard to locate.

## What actually changes

The API call is still small, but the production behavior is platform-specific.

| Check | Android/iOS | Web | macOS/Linux/Windows |
| --- | --- | --- | --- |
| `version`, `buildNumber` | Read from the native bundle/package | Read from generated `version.json` | Read from platform metadata |
| `installerStore` | May identify the store | `null` | `null` |
| Common failure | Build toolchain or stale native build | Missing file or CORS | Platform-specific release configuration |
| Safe fallback | Show an internal “unknown” state | Do not block startup | Treat `null` as a valid result |

The important distinction is that “the package returned `null`” and “the package failed to load” are not the same event. An installer store can legitimately be unknown after an ADB install, on desktop, or on web. A missing web `version.json` is an asset/deployment problem and should be logged separately.

## Upgrade in a reversible order

Change the dependency only after recording the app’s current build matrix. A minimal `pubspec.yaml` change looks like this:

```yaml
dependencies:
  package_info_plus: ^10.2.2
```

Then verify the toolchain before debugging Dart code:

```bash
flutter --version
dart --version
java -version
cd android && ./gradlew --version
```

The package page checked on 2026-11-01 lists Flutter 3.38.1 and Dart 3.10 as minimums, plus Java 17, AGP 8.12.1, and Gradle 8.13 for the current release. Those are package requirements, not a promise that every older project can upgrade without other Android changes.

Keep the migration easy to bisect:

1. Upgrade Flutter/Dart and Android build tools in a dedicated branch.
2. Run `flutter pub get` and a debug build on each target platform.
3. Upgrade `package_info_plus`.
4. Test the metadata reader with a release-like build, not only hot reload.

## Make metadata failure non-fatal

Do not call `PackageInfo.fromPlatform()` before Flutter has initialized its bindings. Also, do not make the first screen wait forever for an analytics label or an “about” dialog value.

```dart
import 'package:flutter/widgets.dart';
import 'package:package_info_plus/package_info_plus.dart';

Future<PackageInfo?> loadPackageInfo() async {
  try {
    return await PackageInfo.fromPlatform();
  } catch (error, stackTrace) {
    debugPrint('package metadata unavailable: $error');
    debugPrintStack(stackTrace: stackTrace);
    return null;
  }
}

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final packageInfo = await loadPackageInfo();
  runApp(App(packageInfo: packageInfo));
}
```

The `null` result here is an application fallback, not a replacement for fixing a broken release pipeline. Use it to keep the UI available while the error reaches telemetry.

## iOS: rebuild native metadata after changing the version

If `pubspec.yaml` contains a new version but iOS still reports the old value, a Dart-only rebuild can preserve stale Xcode build artifacts. The package documentation specifically calls out cleaning the Xcode build folder after changing the version string.

```text
Product -> Clean Build Folder
```

After cleaning, build the same configuration that will be distributed. A simulator run is useful, but it does not prove that the archive contains the expected bundle version. Also keep iOS versions in Apple’s accepted numeric-and-dot format; marketing labels such as `1.4.0-beta` belong in a separate app-level field.

## Web: treat `version.json` like a deployed asset

On Flutter Web, `package_info_plus` looks for a generated `version.json`. A successful Dart compilation does not guarantee that the file is reachable from the deployed base URL. A reverse proxy, a sub-path deployment, or restrictive CORS headers can break only this metadata request.

Add a release check that opens the final URL and verifies:

```bash
curl -i https://example.com/version.json
```

The response should be reachable from the browser origin and contain the version fields expected by the package. If the app is hosted below `/app/`, confirm the generated asset path and the `baseUrl` passed to `PackageInfo.fromPlatform()` match the deployment layout. Do not “fix” a CORS failure by hard-coding a version in Dart; that hides the broken release artifact.

## A small release checklist

- [ ] Flutter/Dart and Android build requirements meet the package’s current minimums.
- [ ] `WidgetsFlutterBinding.ensureInitialized()` runs before the platform call.
- [ ] Android release and iOS archive report the intended version and build number.
- [ ] iOS was rebuilt after a version change, including a clean Xcode build folder when needed.
- [ ] Web `version.json` returns successfully from the production URL.
- [ ] `installerStore == null` is accepted for web, desktop, simulator, and manual installs.
- [ ] Metadata failure does not prevent the app from rendering.

The practical migration rule is simple: upgrade the build environment separately, then validate native metadata per platform. `package_info_plus` removes platform-channel boilerplate, but it cannot repair stale Xcode artifacts, missing web assets, or an installer that does not identify itself. Those boundaries are where the migration either becomes predictable or turns into a release-only bug.

Sources: [package_info_plus package documentation](https://pub.dev/packages/package_info_plus), [package_info_plus changelog](https://pub.dev/packages/package_info_plus/changelog).
