---
layout: post
title: "firebase_messaging Background Handler - Fix Missing Flutter Notifications in Release Builds"
description: "Fix firebase_messaging background messages that work in debug but disappear in Flutter release builds by separating isolate, tree-shaking, and platform checks."
date: 2026-09-20
tags: [firebase_messaging, networking, Android, iOS, Web, debugging]
comments: true
share: true
---

![Firebase Messaging background handler flow](https://firebase.google.com/static/images/brand-guidelines/logo-logomark.png)

If `firebase_messaging` receives notifications in debug but stops calling your handler in a release APK, the first fix is not a longer timeout or another stream listener. Make the handler a top-level entry point, keep Firebase initialization inside that background execution context, and test notification and data payloads separately. This applies to Android and Apple builds; Flutter Web needs a service worker instead, so the mobile fix does not apply there.

## Why the debug build hides the bug

On Android, the background callback runs in a separate isolate. It does not share the `Firebase.initializeApp()` call, service locator, `BuildContext`, or UI state from the main isolate. In release mode, tree shaking can also remove a private callback that is only referenced by the native plugin. The result looks like an FCM delivery failure even when the device received the message.

| Symptom | Likely boundary | Check |
| --- | --- | --- |
| Debug works, release does nothing | Tree shaking | `@pragma('vm:entry-point')` is directly above the callback |
| Handler throws `No Firebase App` | New isolate | Call `Firebase.initializeApp()` inside the callback |
| Handler tries to update a widget | UI is unavailable | Persist data or sync it when the app returns to foreground |
| Mobile fix has no effect on Web | Different runtime | Register `firebase-messaging-sw.js` and its scope |

## A handler that survives release builds

Keep this function outside every class. The annotation placement matters: it must be attached to the function that `onBackgroundMessage` receives.

```dart
import 'package:flutter/widgets.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';

@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();

  // Do short, isolate-safe work here: write a small record, refresh a cache,
  // or call a service that does not depend on BuildContext.
  final messageId = message.messageId;
  final data = message.data;
  print('Background message: $messageId, $data');
}

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  FirebaseMessaging.onBackgroundMessage(firebaseMessagingBackgroundHandler);
  runApp(const MyApp());
}
```

The callback registration belongs in `main`, but its dependencies must be safe in a new isolate. Do not read a Riverpod provider that is only mounted in the widget tree, call a navigator, or assume a singleton initialized by `runApp`. If the background work needs a database, open that database in the handler and close or reuse it according to that package's isolate rules.

## Notification payload and data payload are different tests

A notification payload can be displayed by the operating system while the app is backgrounded, so seeing a tray notification does not prove that your Dart callback ran. Send a data-only message as a separate test and verify the persisted result after reopening the app. Also test a terminated Android app, because a foreground test exercises `onMessage`, not the background path.

The handler should finish quickly. Firebase's Flutter documentation warns that long-running work can be terminated after roughly 30 seconds. Store the minimum event needed to reconcile state later; do not turn the callback into a full synchronization job.

## Web needs a different repair

For Flutter Web, create `web/firebase-messaging-sw.js`, load the Firebase app and messaging compatibility scripts, initialize the same Firebase configuration, then register the worker from the page bootstrap. The worker owns background delivery; `onBackgroundMessage` in Dart is not a substitute. Confirm the worker scope and configuration in browser DevTools after a production deployment, because a stale service worker can keep serving an older configuration.

## Release checklist

- [ ] Callback is top-level and has `@pragma('vm:entry-point')`.
- [ ] `Firebase.initializeApp()` runs inside the callback before other Firebase services.
- [ ] Background code does not touch UI state or `BuildContext`.
- [ ] Notification, data-only, foreground, background, and terminated cases are tested separately.
- [ ] Android release and iOS capability requirements are checked independently.
- [ ] Web uses and registers `firebase-messaging-sw.js`.

The practical boundary is simple: the main isolate owns UI, the background callback owns short-lived persistence, and Web owns background delivery through a service worker. Checking those boundaries explains most “works in debug, missing in release” reports without adding another notification package.

Official references: [Receive messages in Flutter](https://firebase.google.com/docs/cloud-messaging/flutter/receive-messages) and [Firebase Cloud Messaging Flutter codelab](https://firebase.google.com/codelabs/firebase-fcm-flutter).
