---
layout: post
title: "flutter_secure_storage Android Backup Restore - Prevent InvalidKeyException After App Migration"
description: "Fix flutter_secure_storage failures after Android backup restore by separating backup rules from encryption migration and adding a recovery check."
date: 2026-09-19
tags: [flutter_secure_storage, Android, security, migration, networking]
comments: true
share: true
---

![Android phone showing a secure storage and backup configuration](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?auto=format&fit=crop&w=1600&q=80)

If a Flutter app uses `flutter_secure_storage` for tokens or device secrets, the safest Android default is to exclude its encrypted preferences from backup. A restored app can receive encrypted values without the Keystore key that created them, producing `InvalidKeyException`, empty reads, or a write that appears to succeed but cannot be read later. This fix is for apps where the secret can be recreated after sign-in. It is not enough for a wallet or recovery-key app that must preserve the secret across devices; that case needs an explicit migration and recovery design.

## Why the failure appears after restore

`flutter_secure_storage` stores encrypted data and a platform key. Android Auto Backup normally copies most app data, while the Keystore key is device-bound. After a device transfer, reinstall, or backup restore, the preferences file and the key can describe different encryption states.

The symptom is easy to misdiagnose as a package or ProGuard problem:

| Situation | Likely result | Correct decision |
| --- | --- | --- |
| Access token can be recreated | Old value is unusable after restore | Exclude secure-storage preferences and sign in again |
| Secret must survive an algorithm upgrade | Migration can fail halfway | Enable `migrateWithBackup` and test rollback |
| Secret must move to a new device | Device-bound storage is the wrong transport | Use a server-side recovery flow or user-held export |

The package documentation explicitly warns that Android backup can cause `java.security.InvalidKeyException: Failed to unwrap key`. Android 12 and newer also use a different `data-extraction-rules` format for apps targeting API 31 or later, so adding only the old XML file is an incomplete fix.

## Exclude the encrypted preferences

For an app that can issue a new token, add both backup-rule formats. The exact preference path is the package’s `FlutterSecureStorage` shared-preferences file.

For Android 11 and lower, create `android/app/src/main/res/xml/backup_rules.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<full-backup-content>
    <exclude domain="sharedpref" path="FlutterSecureStorage" />
</full-backup-content>
```

For Android 12 and newer, create `android/app/src/main/res/xml/data_extraction_rules.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<data-extraction-rules>
    <cloud-backup>
        <exclude domain="sharedpref" path="FlutterSecureStorage" />
    </cloud-backup>
    <device-transfer>
        <exclude domain="sharedpref" path="FlutterSecureStorage" />
    </device-transfer>
</data-extraction-rules>
```

Reference both files from the application node. Keep any existing backup attributes and merge the entries rather than replacing unrelated rules.

```xml
<application
    android:name="${applicationName}"
    android:label="my_app"
    android:icon="@mipmap/ic_launcher"
    android:fullBackupContent="@xml/backup_rules"
    android:dataExtractionRules="@xml/data_extraction_rules">
</application>
```

The result is a deliberate logout after restore, not a corrupted secure store. Treat a missing token as an authentication state and route the user to sign-in; do not silently copy the old value into ordinary preferences.

## Separate backup protection from package migration

`migrateWithBackup` solves a different problem. It protects an in-place migration between encryption algorithms by keeping temporary `_BACKUP` values and progress markers. It does not make a device-bound Keystore key portable.

```dart
import 'package:flutter/services.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

final secureStorage = FlutterSecureStorage(
  aOptions: AndroidOptions(
    migrateWithBackup: true,
    // Keep the package's current defaults unless a tested migration requires
    // an explicit algorithm choice.
  ),
);

Future<String?> readAccessToken() async {
  try {
    return await secureStorage.read(key: 'access_token');
  } on PlatformException catch (error, stackTrace) {
    // Send a redacted event to telemetry. Never log the token or key material.
    reportSecureStorageFailure(error.code, stackTrace);
    return null;
  }
}
```

Use this option when upgrading a deployed app from an older `flutter_secure_storage` configuration and the data must survive the algorithm change. Test an interrupted migration, an app restart, and a failed decryption. If the value is only a session token, excluding it from backup is simpler and has less recovery surface.

## Release checklist

- Install an older release, write a token, then upgrade to the new release.
- Test Android 11 rules and Android 12+ rules separately.
- Test device-to-device transfer and cloud restore; they are not identical paths.
- Verify that a restored app shows sign-in instead of an endless retry loop.
- Confirm that logs contain error codes only, never tokens or encrypted blobs.
- If enabling migration, kill the app during migration and verify the next launch.

The key distinction is portability. Backup exclusion prevents an invalid encrypted copy from returning to the app. `migrateWithBackup` reduces data-loss risk while the same installation changes encryption format. Neither option turns Android Keystore data into a cross-device backup format.

Sources checked on 2026-09-19: [flutter_secure_storage Android backup and migration notes](https://pub.dev/packages/flutter_secure_storage), [Android Auto Backup rules](https://developer.android.com/identity/data/autobackup), and [the package issue showing failures after backup restore](https://github.com/juliansteenbakker/flutter_secure_storage/issues/853).
