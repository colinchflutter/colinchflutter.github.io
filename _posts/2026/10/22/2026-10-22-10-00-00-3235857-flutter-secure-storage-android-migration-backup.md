---
layout: post
title: "flutter_secure_storage Android Migration - Keep Encrypted Data Recoverable During Cipher Upgrades"
description: "A production checklist for migrating flutter_secure_storage on Android, including migrateWithBackup, storageNamespace, backup exclusions, and rollback checks."
date: 2026-10-22
tags: [flutter_secure_storage, migration, Android, security, recovery, Flutter]
comments: true
share: true
---

![Flutter secure storage Android migration and encrypted data recovery](https://images.unsplash.com/photo-1563013544-824ae1b704d3?auto=format&fit=crop&w=1600&q=80)

The safe choice for a Flutter app upgrading `flutter_secure_storage` on Android is to treat the cipher change as a data migration, not a dependency bump. Enable `migrateWithBackup: true` for critical values, keep one stable `storageNamespace`, and test an in-place upgrade from the oldest supported build. This applies to apps that already have encrypted data; a clean install has nothing to migrate.

## The failure boundary

`flutter_secure_storage` 10 introduced new Android cipher defaults. The changelog says old `RSA_ECB_PKCS1Padding` and `AES_CBC_PKCS7Padding` data can migrate to the newer RSA-OAEP and AES-GCM defaults when `migrateOnAlgorithmChange` is enabled. The package documentation also states that `migrateWithBackup` is disabled by default for backward compatibility.

That default matters. A migration can run while the process is killed, the device loses power, or two storage instances use different configuration. A failed read must not immediately trigger `deleteAll()` or a fresh token write. Those actions can turn a recoverable migration problem into permanent logout or secret loss.

| Situation | Safer decision |
| --- | --- |
| Ordinary session token | Migrate with backup, then allow re-login as a fallback |
| Wallet seed, recovery code, or device credential | Stop rollout and keep an explicit recovery path |
| Multiple storage instances | Give each instance a deliberate `storageNamespace` |
| Restored Android backup | Exclude secure-storage preferences or disable backup |

The picture to keep in mind is simple: the encrypted value, key metadata, and Android backup boundary must move together.

## A migration-safe configuration

Pin the package through a reviewed lockfile, then make the migration policy explicit in the one place that constructs storage.

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

const androidOptions = AndroidOptions(
  migrateOnAlgorithmChange: true,
  migrateWithBackup: true,
  storageNamespace: 'app_credentials',
);

final secureStorage = FlutterSecureStorage(
  aOptions: androidOptions,
);
```

`storageNamespace` is preferable to reusing the older `sharedPreferencesName` when more than one store exists. It isolates the SharedPreferences file, KeyStore aliases, and configuration for that store. Do not change the namespace at the same time as the cipher migration unless the change itself is covered by a planned data move; otherwise an empty store can look like deleted credentials.

## Android backup is a separate trap

The package documentation warns that restoring encrypted SharedPreferences onto another Android installation can produce `InvalidKeyException: Failed to unwrap key`. The backup contains data that the destination KeyStore cannot unwrap. Choose one policy and verify it in a restore test:

```xml
<application
    android:allowBackup="true"
    android:fullBackupContent="@xml/backup_rules"
    android:dataExtractionRules="@xml/data_extraction_rules">
</application>
```

The XML rules should exclude the preferences used by `flutter_secure_storage`. If the app does not need Android backup at all, `android:allowBackup="false"` is simpler, but it changes the recovery experience for every other preference too. The right choice depends on whether the app can recreate its non-secret state.

## Reproducible release checklist

Run this matrix before shipping the upgrade. The important comparison is an existing install, not `flutter run` after clearing data.

- Install the oldest supported release and write a representative token set.
- Upgrade in place to the new build; confirm every key is readable before writing it again.
- Kill the app during startup and repeat the read; verify the backup migration resumes.
- Test a fresh install, an app-data restore, and a namespace change as separate cases.
- Record `PlatformException` code and package version without logging secret values.
- Keep the previous build available until the upgrade cohort has a recovery path.

For a non-critical session token, a controlled sign-in screen can be the final fallback. For an irreplaceable secret, “ask the user to log in again” is not recovery. The migration must preserve the original value or provide an independently stored recovery mechanism.

`flutter_secure_storage` is a storage boundary, so the version number alone is not the test plan. Verify the resolved package version, the actual Android options, backup rules, and the oldest on-device data shape together. The package documentation and changelog were checked on 2026-10-22.

Sources: [flutter_secure_storage documentation](https://pub.dev/packages/flutter_secure_storage), [flutter_secure_storage changelog](https://pub.dev/packages/flutter_secure_storage/changelog), and [Android backup documentation](https://developer.android.com/identity/data/autobackup).
