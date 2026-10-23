---
layout: post
title: "firebase_messaging 17 Migration - Keep Flutter Push Delivery Compatible Across Android and iOS"
description: "A production checklist for migrating Flutter firebase_messaging 17.0.0, covering Android minSdk 24, Firebase Apple SDK 13, background-engine bursts, and duplicate notification taps."
date: 2026-10-24
tags: [firebase_messaging, migration, notifications, Android, iOS, testing, Flutter]
comments: true
share: true
---
![firebase_messaging 17 migration and Flutter push delivery flow](/assets/images/firebase-messaging-notification-tap-deduplication.png)

If a Flutter app can drop Android 23 and already follows the current Firebase toolchain, `firebase_messaging` 17.0.0 is a reasonable upgrade. If the app still supports Android API 23, ships a macOS target below 11 through Swift Package Manager, or cannot update the rest of its FlutterFire set, keep the old major temporarily and schedule the platform migration first. The package release is not just a Dart dependency bump.

## What changed at the upgrade boundary

The 17.0.0 changelog records four changes that affect production behavior: Firebase Apple SDK 13, Android Firebase BoM 35 with `minSdk 24`, one background Flutter engine for bursts of Android messages, and delivery of a tap on the push that launched the iOS app in the background. The FlutterFire release notes place `firebase_messaging 17.0.0` in the 5.0.0 BoM with `firebase_core 5.0.0` and Flutter `>=3.27.0`.

| Check | Before changing code | Migration decision |
|---|---|---|
| Android floor | `minSdk` 21 or 23 | Raise to 24, or remain on 16.x |
| Apple dependencies | CocoaPods or Swift Package Manager | Resolve Firebase Apple SDK 13 with the chosen integration |
| Background data | One engine assumed per callback | Make the handler idempotent and storage-safe |
| Notification tap | `getInitialMessage` and stream both navigate | Route through one deduplicating handler |

This is the answer gap in many upgrade notes: a project may compile after changing `pubspec.yaml`, yet its supported-device list, native dependency manager, and message state machine can still disagree.

## Upgrade the FlutterFire set together

Pin the related Firebase packages to a compatible release line instead of upgrading only messaging. The exact versions should come from the FlutterFire BoM resolved for the project, not from an old code snippet.

```yaml
dependencies:
  firebase_core: ^5.0.0
  firebase_messaging: ^17.0.0
```

Then inspect the resolved graph and native floors before committing the lockfile:

```bash
flutter pub outdated
flutter pub deps --style=compact | rg 'firebase_(core|messaging)'
flutter build apk --release
flutter build ios --release --no-codesign
```

For Android, verify `android/app/build.gradle` or `build.gradle.kts` resolves `minSdk` to 24 or higher. Do not “fix” a lower device floor by forcing an older Firebase Android BoM: that can produce a build that passes locally while the plugin and native SDK no longer share the tested boundary.

On iOS, update the native Firebase dependency through the integration your project actually uses. A Swift Package Manager target that needs notification-service-extension code must include `FirebaseMessaging` in that extension target too; adding it only to `Runner` does not make the extension capable of processing the payload. Firebase’s current receive-messages guide documents this target-level setup.

## Keep background handling safe after the Android engine fix

The new release fixes a burst case inside the plugin, but application code still has to tolerate repeated delivery, process restarts, and a background isolate without widgets. Register a top-level handler, initialize Firebase in that execution context, and write a deduplication key before performing a side effect.

```dart
@pragma('vm:entry-point')
Future<void> handleBackgroundMessage(RemoteMessage message) async {
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  final eventId = message.messageId;
  if (eventId == null || await eventAlreadyHandled(eventId)) return;

  await markEventHandled(eventId);
  await persistInboxItem(message.data);
}

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  FirebaseMessaging.onBackgroundMessage(handleBackgroundMessage);
  runApp(const App());
}
```

`eventAlreadyHandled` and `markEventHandled` must use storage available to the background context. A provider mounted below `runApp`, a `BuildContext`, or a UI-only singleton is not a valid dependency here. The `messageId` check is a policy example, not a promise that every upstream message has a stable ID; if the server can omit it, send an application event ID in `data`.

## Make notification taps one navigation event

A terminated launch and a background tap are different delivery paths. Treat both as input to one idempotent route function. The 17.0.0 release improves iOS delivery for the push that launched the app in the background, so code that assumed only one path can now expose duplicate navigation during migration.

```dart
final handledTapIds = <String>{};

void consumeTap(RemoteMessage? message) {
  if (message == null) return;
  final id = message.messageId ?? message.data['event_id'];
  if (id == null || !handledTapIds.add(id)) return;

  final route = message.data['route'];
  if (route != null) appNavigator.go(route);
}

Future<void> installTapHandlers() async {
  consumeTap(await FirebaseMessaging.instance.getInitialMessage());
  FirebaseMessaging.onMessageOpenedApp.listen(consumeTap);
}
```

For a cold start, delay `consumeTap` until the navigator is ready. For a real app, persist the handled event ID if the same notification must not reopen a screen after process recreation. The in-memory set only prevents duplicates during one process lifetime.

## Release checklist

- [ ] Android API 23 is either removed from the support matrix or the package major is held back.
- [ ] `firebase_core` and other FlutterFire packages resolve from one compatible BoM line.
- [ ] iOS/macOS native Firebase dependencies resolve with the project’s package manager.
- [ ] Notification-service extensions contain `FirebaseMessaging` when they process payloads.
- [ ] Background code has no widget-tree dependency and can run twice safely.
- [ ] A terminated tap and a background tap reach the same idempotent route handler.
- [ ] Tests cover Android message bursts, iOS cold launch, Android 13+ notification denial, and an app process restart.

The practical migration rule is to treat `firebase_messaging 17` as a platform-and-delivery change. Apps with the required platform floors gain the current fixes, while apps with older devices or hand-written native notification paths should stage the native upgrade and message-state tests before changing the package constraint.

Sources checked 2026-10-24: [firebase_messaging changelog](https://pub.dev/packages/firebase_messaging/changelog), [Firebase SDK for Flutter release notes](https://firebase.google.com/support/release-notes/flutter), [receive messages in Flutter](https://firebase.google.com/docs/cloud-messaging/flutter/receive-messages), and [FCM Flutter setup](https://firebase.google.com/docs/cloud-messaging/flutter/get-started).
