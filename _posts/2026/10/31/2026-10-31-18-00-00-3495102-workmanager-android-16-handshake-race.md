---
layout: post
title: "workmanager Android 16 Handshake Race - Recover Flutter Background Tasks That Hang"
description: "Fix workmanager background jobs that hang or fail after about 30 seconds on Android 16 by upgrading the handshake fix, preserving the dispatcher entry point, and checking plugin initialization."
date: 2026-10-31
tags: [workmanager, Android, background, migration, testing, Flutter]
comments: true
share: true
---

![Flutter workmanager Android 16 background task handshake and Dart isolate startup](https://images.unsplash.com/photo-1518770660439-4636190af475?auto=format&fit=crop&w=1600&q=80)

If a Flutter `workmanager` job reaches Android 16 but hangs for roughly 30 seconds, upgrade to the release containing the Dart-initiated background handshake before rewriting the task. This fix is for Android 16 projects using `workmanager` 0.10.9 or an affected `workmanager_android` dependency. It does not make periodic work run at an exact time, and it does not repair a dispatcher removed by release-mode tree shaking.

The important failure is a race, not simply “Android killed my worker”. In the affected path, native code could send the task before the background Dart isolate had registered its channel handlers. The message was lost, so the callback never ran; the watchdog then turned the silent loss into a timeout. The 0.10.10 release reverses the order: Dart registers the handlers first, then signals native code that the channel is ready.

| Symptom | Likely boundary | Action |
| --- | --- | --- |
| About 30-second timeout on Android 16 | Background handshake race | Move to `workmanager` 0.10.10 or later |
| Callback is never found in release | Tree shaking removed the entry point | Keep a top-level dispatcher and `@pragma('vm:entry-point')` |
| `channel-error` inside the task | Plugin is not registered in the background engine | Initialize the plugin or registrant inside the dispatcher |
| Task runs but is still delayed | Android scheduling constraints | Check constraints, battery policy, and WorkManager timing |

The first check is the resolved dependency, not only `pubspec.yaml`. A transitive Android implementation can remain older than the version you think you selected.

```yaml
dependencies:
  workmanager: ^0.10.10
```

```bash
flutter pub deps --style=compact | rg "workmanager"
flutter pub get
```

The dispatcher must remain reachable from a background engine. Keep it top-level, annotate it, and initialize dependencies in that isolate. The exact plugin setup can vary by package version, but the entry-point requirements do not.

```dart
import 'package:flutter/widgets.dart';
import 'package:workmanager/workmanager.dart';

@pragma('vm:entry-point')
void backgroundDispatcher() {
  Workmanager().executeTask((taskName, inputData) async {
    WidgetsFlutterBinding.ensureInitialized();

    switch (taskName) {
      case 'sync-cache':
        // Create background-safe clients here, not by reusing UI objects.
        await syncCache(inputData);
        return true;
      default:
        return false;
    }
  });
}

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Workmanager().initialize(backgroundDispatcher);
  runApp(const App());
}
```

If `syncCache` uses `shared_preferences`, a database, Firebase, or another platform plugin, initialize that dependency in the background path according to its own documentation. A singleton created in `main()` belongs to the UI isolate; it is not automatically available to the worker. Returning `false` asks WorkManager to retry, so only use it for a failure that is safe to repeat. Do not return `true` after catching an upload error just to make the task disappear.

Run a small release-oriented checklist before shipping the upgrade:

- [ ] `flutter pub deps` resolves the fixed `workmanager_android` version.
- [ ] The dispatcher is top-level or static and has `@pragma('vm:entry-point')`.
- [ ] The task name in registration exactly matches the `switch` case.
- [ ] Every plugin used by the worker is initialized in the background engine.
- [ ] A duplicate run is safe after a process death or WorkManager retry.
- [ ] Logs include task name, start time, completion result, and exception class.
- [ ] The test uses a release/profile build on an Android 16 device or emulator.

Do not “fix” this by increasing the timeout or registering the same task repeatedly. Those changes can hide the handshake problem and create duplicate work. Also separate two cases: upgrading to the handshake fix addresses the lost message during isolate startup; a task that starts successfully but exceeds Android’s execution limits needs smaller units of work, resumable state, or an appropriate foreground-service path.

The practical decision is simple: upgrade the resolved Workmanager implementation first, preserve the background entry point, and verify plugin registration in the separate engine. If the callback still does not start, the next useful artifact is the release log around isolate startup—not another scheduling call.

Sources: [workmanager 0.10.10 changelog](https://pub.dev/packages/workmanager/changelog), [the 0.10.10 Android handshake release note](https://github.com/fluttercommunity/flutter_workmanager/releases), [Flutter background processes](https://docs.flutter.dev/packages-and-plugins/background-processes), and [workmanager callback API](https://pub.dev/documentation/workmanager/latest/workmanager/Workmanager-class.html).
