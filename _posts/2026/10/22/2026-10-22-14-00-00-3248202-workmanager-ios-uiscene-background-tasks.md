---
layout: post
title: "workmanager iOS UIScene Migration - Keep Flutter Background Tasks Registered"
description: "Fix workmanager background jobs that stop after Flutter's iOS UIScene migration by restoring launch-handler registration, plugin registration, and identifier checks."
date: 2026-10-22
tags: [workmanager, migration, iOS, background, testing, Flutter]
comments: true
share: true
---

![Flutter workmanager iOS UIScene background task registration flow](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?auto=format&fit=crop&w=1600&q=80)

If `workmanager` jobs stop arriving after an iOS UIScene migration, restore two separate paths: register the persisted `BGTaskScheduler` launch handlers from `AppDelegate`, and register plugins for the background Flutter engine. This applies to iOS apps using `workmanager_apple` with processing or refresh tasks. It does not make iOS run work at an exact time, and it is unnecessary for Android-only jobs.

The failure is easy to misdiagnose because the Dart scheduling call can still succeed. The operating system later launches the app in the background, but the handler was registered too late, or the background engine cannot find a plugin such as `shared_preferences` or Firebase.

## The boundary that changed

Flutter's UIScene migration moves UI-related engine setup away from the old launch path. Flutter's migration guide says plugin registration for the implicit engine belongs in `didInitializeImplicitFlutterEngine`. Apple's `BGTaskScheduler` rule is different: every permitted identifier must have a launch handler registered before application launch finishes, and an identifier cannot be registered twice.

| Responsibility | Correct location after UIScene | Typical symptom when missing |
| --- | --- | --- |
| Re-register saved background task handlers | `application(_:didFinishLaunchingWithOptions:)` | Task launch fails or is never delivered after relaunch |
| Register Flutter plugins for the background engine | `didInitializeImplicitFlutterEngine` plus Workmanager registrant callback | `MissingPluginException` or channel-error |
| Keep task names consistent | Dart, `Info.plist`, and native registration | `BGTaskSchedulerErrorDomain` code 3 |

The important detail is that these are not interchangeable. Moving everything into `didInitializeImplicitFlutterEngine` is too late for Apple's launch-handler requirement.

## AppDelegate.swift fix

Use the Workmanager registrant callback for the separate background engine, while keeping launch-handler restoration in the application launch method.

```swift
import Flutter
import UIKit
import workmanager_apple

@main
@objc class AppDelegate: FlutterAppDelegate, FlutterImplicitEngineDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    // Required for tasks persisted before the app adopted UIScene.
    WorkmanagerPlugin.registerLaunchHandlers()

    // Makes Firebase, shared_preferences, and other safe plugins
    // available inside Workmanager's background Flutter engine.
    WorkmanagerPlugin.setPluginRegistrantCallback { registry in
      GeneratedPluginRegistrant.register(with: registry)
    }

    return super.application(
      application,
      didFinishLaunchingWithOptions: launchOptions
    )
  }

  func didInitializeImplicitFlutterEngine(
    _ engineBridge: FlutterImplicitEngineBridge
  ) {
    GeneratedPluginRegistrant.register(with: engineBridge.pluginRegistry)
  }
}
```

Do not call `registerLaunchHandlers()` again from the scene delegate. Apple documents that registering one identifier twice can terminate the app. The Workmanager method restores the identifiers it persisted; it is not a second scheduling call.

## Dart entry point and identifier check

The callback must survive release-mode tree shaking and initialize dependencies inside the background isolate. Keep the task identifier as one constant so the value cannot drift between scheduling and handling.

```dart
import 'package:flutter/widgets.dart';
import 'package:workmanager/workmanager.dart';

const syncTask = 'com.example.app.sync';

@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((task, inputData) async {
    WidgetsFlutterBinding.ensureInitialized();
    DartPluginRegistrant.ensureInitialized();

    if (task == syncTask) {
      // Initialize Firebase or storage here, not from main().
      await runSync(inputData);
    }
    return true;
  });
}

Future<void> runSync(Map<String, dynamic>? inputData) async {
  // Keep this work UI-free and idempotent.
}
```

The exact Dart API can vary with the Workmanager version, but the constraints do not: top-level or static dispatcher, `@pragma('vm:entry-point')`, one dispatcher registration, and background-safe plugins only. A plugin that requires a view controller or UI isolate is not made safe by registering it.

For a `BGProcessingTask`, the same identifier must appear in the iOS configuration:

```xml
<key>UIBackgroundModes</key>
<array>
  <string>processing</string>
</array>
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
  <string>com.example.app.sync</string>
</array>
```

If the project uses Workmanager's explicit native registration option, register that same string in `AppDelegate` as documented by the package. A short Dart name such as `sync` and a reverse-DNS plist name are not equivalent.

## Migration checklist

- [ ] Flutter app has adopted the UIScene template and `AppDelegate` conforms to `FlutterImplicitEngineDelegate`.
- [ ] `GeneratedPluginRegistrant` moved to `didInitializeImplicitFlutterEngine`.
- [ ] `WorkmanagerPlugin.registerLaunchHandlers()` runs once during application launch.
- [ ] `WorkmanagerPlugin.setPluginRegistrantCallback` is configured before the background engine starts.
- [ ] Dispatcher is top-level and has `@pragma('vm:entry-point')`.
- [ ] The task identifier matches in Dart, `Info.plist`, and any native Workmanager registration.
- [ ] `UIBackgroundModes` and permitted identifiers are present for the selected iOS task type.
- [ ] Background code uses only isolate-safe plugins and handles expiration or retry.

Test on a physical device, because iOS scheduling is best-effort. Schedule the task, terminate the app, let the system launch it, and inspect the native console for registration errors. For development-only testing, Apple's private debugger commands can simulate a launch, but they must not ship in the app. Also test an upgrade from a build that scheduled the old task: that is the case `registerLaunchHandlers()` is meant to preserve.

The practical rule is simple: UIScene changes when Flutter plugins attach to an engine; it does not remove Apple's requirement to register every background task early. Keep those two registration paths explicit, and a background job that worked before the migration has a reproducible recovery path afterward.

Sources: [Flutter UIScene migration guide](https://docs.flutter.dev/release/breaking-changes/uiscenedelegate), [Workmanager quickstart](https://github.com/fluttercommunity/flutter_workmanager/blob/main/docs/quickstart.mdx), and [Apple BGTaskScheduler registration](https://developer.apple.com/documentation/backgroundtasks/bgtaskscheduler/register%28fortaskwithidentifier%3Ausing%3Alaunchhandler%3A).
