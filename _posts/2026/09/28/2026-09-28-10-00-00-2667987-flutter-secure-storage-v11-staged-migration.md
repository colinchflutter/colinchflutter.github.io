---
layout: post
title: "flutter_secure_storage 11 Android Migration - Avoid Direct-Upgrade Data Loss"
description: "A production-safe Flutter migration path from flutter_secure_storage 9 or 10 to 11 on Android, including staged cipher migration, checkUpgradeStatus, and recovery decisions."
date: 2026-09-28
tags: [flutter_secure_storage, migration, Android, security, recovery]
comments: true
share: true
---

![Android phone showing a secure storage and backup configuration](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?auto=format&fit=crop&w=1600&q=80)

`flutter_secure_storage` 11 should not be treated as a normal dependency bump when an Android app still has data written by version 9 or an older configuration. The safer choice is to release a version 10 bridge first, let it migrate the old cipher data, and only then move to version 11. A direct 9 → 11 upgrade is acceptable only when the app has no legacy secure-storage data to preserve, or when losing that data is an explicitly handled outcome such as re-authentication.

## Why the direct upgrade is risky

Version 10 replaced the Android implementation with custom ciphers and changed the defaults to RSA OAEP plus AES-GCM. It also introduced migration from older algorithms. Version 11 removed deprecated algorithms and configuration options, so the old data must already have crossed the version 10 migration boundary. The package changelog explicitly says data written before version 10 can become unusable after a direct upgrade to 11.

The failure is easy to misread. A token read returns `null`, the app sends the user to login, or `resetOnError` deletes the unreadable entries. That is not the same as “the user logged out.” It can be a package migration failure that has already removed the only local refresh token.

| Current production state | Upgrade choice | Reason |
| --- | --- | --- |
| 9.x or older Android cipher | 9.x → 10.x → 11.x | Version 10 is the migration bridge |
| 10.x with migrated default ciphers | 10.x → 11.x after option audit | The removed legacy algorithms are no longer needed |
| No persisted secure-storage data | Direct 11.x upgrade | There is nothing to migrate, but build constraints still apply |
| Wallet seed or unrecoverable secret only in the device | Pause and design recovery first | A package migration cannot recreate lost data |

The version references above are based on the package changelog checked on 2026-09-28. Confirm the exact resolved version in `pubspec.lock` before shipping; a caret constraint alone is not a migration plan.

## The bridge release for 9.x users

The bridge release should still use the version 10 API and the same Android storage namespace that production already uses. Do not combine a namespace rename, biometric policy change, and package-major upgrade in one release. Each change can alter which key or preference file is read.

This is the configuration shape for the bridge. The option names are intentionally explicit so a future diff shows what changed.

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

const bridgeAndroidOptions = AndroidOptions(
  // Keep the migration crash-recovery copies during the bridge release.
  migrateWithBackup: true,
  migrateOnAlgorithmChange: true,
  resetOnError: false,
);

final bridgeStorage = FlutterSecureStorage(
  aOptions: bridgeAndroidOptions,
);
```

`resetOnError: false` is a rollout safety choice for the bridge. If decryption fails, keeping the bytes on disk gives the app a chance to report the failure and use a server-side recovery path instead of silently deleting them. It does not repair corrupted data. The package documentation describes `migrateWithBackup` as a crash-resistant migration aid, not as a way to decrypt data when the old key or algorithm is already unavailable.

Ship this bridge to a small percentage first and record only operational metadata: package version, migration result, platform, and whether re-authentication was required. Never log the key names, token values, encrypted payloads, or exception text if it contains storage identifiers.

After the bridge has been installed and opened successfully, verify that the old data is readable and that the server can refresh the session. Only then prepare the version 11 release.

## The version 11 guard before the first read

Version 11.1 added `checkUpgradeStatus()`. It returns a read-only `SecureStorageUpgradeStatus`; according to the API documentation it does not write, migrate, or delete anything. That makes it useful before the first normal `read` or `write` in an upgrade-aware startup path.

```dart
Future<void> prepareSecureStorage() async {
  const options = AndroidOptions(
    migrateWithBackup: true,
    resetOnError: false,
  );
  final storage = FlutterSecureStorage(aOptions: options);
  final status = await storage.checkUpgradeStatus(aOptions: options);

  if (status.hasDataLoss || status.willDiscardOnNextAccess) {
    // Do not call read() yet. Keep the failure recoverable.
    await reportStorageUpgradeFailure(
      state: status.state.toString(),
      reason: status.reason.toString(),
      entries: status.entryCount,
    );
    return;
  }

  final refreshToken = await storage.read(key: 'refresh_token');
  await continueWithSession(refreshToken);
}
```

The method is a detection boundary, not a migration command. `willDiscardOnNextAccess` is especially important when `resetOnError` is enabled: the next access may delete unreadable legacy data. In a normal account-based app, the recovery branch can invalidate the local session, show login, and fetch the account state again after authentication. For a device-bound secret, that branch must stop and ask for an explicit recovery key or server-assisted restoration.

Keep this guard in the first release that adopts version 11. It gives telemetry a chance to distinguish “no token exists” from “the token became unreadable during a package upgrade.” Remove it only after the oldest supported install path has aged out and the product no longer needs to diagnose that migration.

## Reproduce the upgrade before publishing

A clean install is not a migration test. Use an emulator or test device and preserve the same application ID, signing configuration, storage namespace, and Android options as production.

```text
[ ] Install the production 9.x build.
[ ] Write a non-sensitive sentinel and a test refresh token.
[ ] Force-stop and relaunch; verify both values are readable.
[ ] Install the version 10 bridge over the existing app.
[ ] Launch twice, including one launch after process death.
[ ] Verify the sentinel, session refresh, and migration telemetry.
[ ] Install version 11 over the bridged app.
[ ] Run checkUpgradeStatus before the first read or write.
[ ] Test sign-out, reinstall, backup restore, and a failed network refresh separately.
```

The sentinel should be disposable and the test token should be revoked after the run. Do not use a real user credential to prove migration. Also test the negative path by installing 11 directly over 9.x. The expected result is not “the app must recover”; it is that the app detects the unsupported path without silently presenting a fresh account or deleting data before telemetry runs.

## Changes that belong in the same review

Version 11 also removes `encryptedSharedPreferences` and `sharedPreferencesName` from `AndroidOptions`, replaces the latter with `storageNamespace`, and raises Android requirements. Search the whole project, including flavor-specific files and generated configuration, before changing the dependency.

| Review item | Decision to record |
| --- | --- |
| `encryptedSharedPreferences` still appears | Complete the v10 migration first; do not delete it blindly |
| `sharedPreferencesName` is used | Map it to the intended `storageNamespace` and keep the namespace stable |
| `minSdk` is below 24 | Raise it only with a product decision about excluded devices |
| `compileSdk` is below the package requirement | Update the Android toolchain in a separate, reproducible build change |
| Secret can be recreated from the server | Re-authenticate or re-fetch after confirmed data loss |
| Secret cannot be recreated | Do not ship until recovery is tested end to end |

The practical rule is simple: migrate data before removing the code that understands it. `migrateWithBackup` reduces the risk of an interrupted migration, while `checkUpgradeStatus` reduces the risk of hiding a failed one. Neither option makes an unrecoverable device-only secret portable.

Sources: [`flutter_secure_storage` changelog](https://github.com/juliansteenbakker/flutter_secure_storage/blob/develop/flutter_secure_storage/CHANGELOG.md), [package migration notes](https://pub.dev/packages/flutter_secure_storage), [`FlutterSecureStorage.checkUpgradeStatus` API](https://pub.dev/documentation/flutter_secure_storage/latest/flutter_secure_storage/FlutterSecureStorage-class.html), [`SecureStorageUpgradeStatus` API](https://pub.dev/documentation/flutter_secure_storage/latest/flutter_secure_storage/SecureStorageUpgradeStatus-class.html), and [reproducible migration issue #1166](https://github.com/juliansteenbakker/flutter_secure_storage/issues/1166).
