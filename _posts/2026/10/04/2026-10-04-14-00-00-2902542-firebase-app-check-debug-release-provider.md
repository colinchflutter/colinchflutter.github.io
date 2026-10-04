---
layout: post
title: "firebase_app_check Debug vs Release - Stop Shipping Flutter App Check Tokens to Production"
description: "A reproducible firebase_app_check setup for Flutter local, CI, and Play Store builds, including provider selection, token safety, and enforcement rollout."
date: 2026-10-04
tags: [firebase_app_check, security, Android, testing, migration]
comments: true
share: true
---
![firebase_app_check provider split between local CI, Play Store release, and backend enforcement](../../../assets/images/firebase-app-check-debug-release-provider.png)

The diagram keeps debug tokens on the development path and Play Integrity on the release path.

The safest `firebase_app_check` setup is not one provider for every Flutter build. Use the debug provider only for local or CI builds, use Play Integrity for an Android Play Store release, and enable enforcement only after checking the App Check metrics. This does not apply if your app has no Firebase resource worth protecting, or if every build is an emulator-only internal prototype.

## The failure that looks like a Firebase outage

An app can work on an emulator and then fail after enabling enforcement. The two builds are not making the same attestation decision:

| Build or environment | Provider | Token handling | Expected use |
| --- | --- | --- | --- |
| Local emulator | `AndroidDebugProvider` | Register a private debug token | Development only |
| CI integration test | Debug provider | Inject a token from CI secrets | Automated test only |
| Android Play release | `AndroidPlayIntegrityProvider` | No debug token in the app | Production |
| Firebase console | Enforcement | Reject invalid or missing tokens | Roll out after metrics review |

The dangerous shortcut is leaving `AndroidProvider.debug` in a shared initialization function. A release APK can then contain a debug path, and a registered debug token is intentionally accepted from an otherwise unverified device. Firebase explicitly warns not to commit that token or ship the debug provider in production.

## Keep provider selection at the build boundary

The current `firebase_app_check` API exposes provider classes such as `AndroidDebugProvider` and `AndroidPlayIntegrityProvider`. Older examples use the enum-style `androidProvider` parameter; check the package version before copying an example because the newer provider arguments are the migration direction.

Put the choice behind a compile-time flag so a release build cannot silently inherit the developer setting. The following is a reproducible shape; the token is supplied outside source control.

```dart
import 'package:firebase_app_check/firebase_app_check.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/foundation.dart';

Future<void> initializeFirebaseProtection() async {
  await Firebase.initializeApp();

  final isDebugBuild = !kReleaseMode;
  final androidProvider = isDebugBuild
      ? const AndroidDebugProvider()
      : const AndroidPlayIntegrityProvider();

  await FirebaseAppCheck.instance.activate(
    providerAndroid: androidProvider,
  );
}
```

Call this before Firestore, Storage, Functions, or another protected Firebase service is first used. On Android, the debug provider prints a token on first run; register that token in Firebase Console for the matching app. Keep it in the console and CI secret store, not in Dart source. Windows is a separate case: the current package example requires an explicit `WindowsDebugProvider(debugToken: ...)`. For a local Windows run, pass that value through a non-committed launch configuration:

```bash
flutter run -d windows --dart-define=APP_CHECK_DEBUG_TOKEN="$APP_CHECK_DEBUG_TOKEN"
```

The release command must not receive that variable. A stronger CI rule is to fail the build if `APP_CHECK_DEBUG_TOKEN` is present while `flutter build apk --release` is running. The exact provider API can vary by package version, so treat the installed package API as authoritative rather than mixing enum and provider-class examples.

## Android release has two independent setup points

Selecting Play Integrity in Dart is only one half. Register the exact Firebase Android app and its signing certificate in App Check. The certificate must match the distribution path: a Play App Signing certificate, an internal testing configuration, or a separately distributed APK may not use the same fingerprint.

Firebase's Android guide also distinguishes Play Store-only distribution from apps distributed both inside and outside Google Play. The acceptable Play Integrity verdict settings must match that choice. A release installed outside Play can be rejected if the console policy requires a Play-recognized installation.

Use this short release checklist:

- [ ] `providerAndroid` is `AndroidPlayIntegrityProvider()` for release.
- [ ] No debug token is stored in Dart, `--dart-define` defaults, logs, or a public CI artifact.
- [ ] Every Firebase Android app variant that will receive traffic is registered.
- [ ] The signing SHA-256 fingerprint belongs to the actual release path.
- [ ] App Check metrics are observed before enforcement is enabled.
- [ ] A fresh install and an upgrade are tested against the protected Firebase operation.
- [ ] A failed attestation shows a recoverable error instead of a permanent loading screen.

## Roll out enforcement without locking out users

Install the new client with App Check active while enforcement is still off. Check request metrics for Firestore, Storage, Realtime Database, Authentication, or Functions—the products your app actually uses. Then enable enforcement for one product or a limited release path first. If valid users disappear from the metrics, stop the rollout and verify the app registration, signing fingerprint, provider, and distribution channel before changing the Dart code.

Do not “fix” a production rejection by registering the production APK as a debug token. That makes the symptom disappear while weakening the boundary App Check is supposed to provide. For local development, revoke a leaked token in Firebase Console and issue a new one. For production, keep the Play Integrity path and correct the console configuration.

The practical rule is simple: debug tokens belong to a private development lane, Play Integrity belongs to the release lane, and enforcement belongs after evidence. Separating those three decisions prevents the common “works on emulator, fails from Play Store” migration loop without requiring a paid service or an affiliate tool.

Sources: [Firebase App Check for Flutter](https://firebase.google.com/docs/app-check/flutter/default-providers), [Flutter debug provider guidance](https://firebase.google.com/docs/app-check/flutter/debug-provider), [Android Play Integrity setup](https://firebase.google.com/docs/app-check/android/play-integrity-provider), and [`firebase_app_check` package API](https://pub.dev/packages/firebase_app_check).
