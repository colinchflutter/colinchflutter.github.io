---
layout: post
title: "file_picker Android SAF URI Grants - Keep Flutter Files Available After Reboot"
description: "Fix file_picker document libraries that lose Android access after a reboot by separating temporary paths from persistable SAF URI grants and migrating the v12 API carefully."
date: 2026-10-17
tags: [file_picker, Android, migration, file_upload, storage]
comments: true
share: true
---

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 210" role="img" aria-label="file_picker Android SAF access flow"><rect width="900" height="210" rx="18" fill="#172033"/><text x="38" y="46" fill="#f8fafc" font-family="Arial,sans-serif" font-size="27" font-weight="700">file_picker: path is not a durable grant</text><rect x="40" y="86" width="190" height="72" rx="12" fill="#334155"/><text x="78" y="129" fill="#e2e8f0" font-family="Arial,sans-serif" font-size="22">Android picker</text><path d="M242 122h105" stroke="#60a5fa" stroke-width="6"/><path d="M338 114l12 8-12 8" fill="none" stroke="#60a5fa" stroke-width="6"/><rect x="365" y="86" width="190" height="72" rx="12" fill="#7c2d12"/><text x="397" y="114" fill="#ffedd5" font-family="Arial,sans-serif" font-size="19">temporary path</text><text x="407" y="140" fill="#ffedd5" font-family="Arial,sans-serif" font-size="16">cache / URI</text><path d="M567 122h105" stroke="#5eead4" stroke-width="6"/><path d="M663 114l12 8-12 8" fill="none" stroke="#5eead4" stroke-width="6"/><rect x="690" y="86" width="170" height="72" rx="12" fill="#0f766e"/><text x="724" y="114" fill="#ccfbf1" font-family="Arial,sans-serif" font-size="18">persistable</text><text x="731" y="140" fill="#ccfbf1" font-family="Arial,sans-serif" font-size="17">SAF handle</text></svg>

The production choice is simple: use an ordinary `PlatformFile` path for a one-time upload, but use a persistable Android SAF handle for a document or folder that the app must reopen after a reboot. This applies to Android document libraries and import/export features; it does not justify `MANAGE_EXTERNAL_STORAGE`, and it is unnecessary when the app copies the selected bytes into its own durable storage immediately.

![The path and the long-lived SAF grant are different contracts](data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 900 120'%3E%3Crect width='900' height='120' fill='%23eef2ff'/%3E%3Ctext x='35' y='48' font-family='Arial' font-size='24' fill='%231e293b'%3EPlatformFile.path = convenient local access%3C/text%3E%3Ctext x='35' y='88' font-family='Arial' font-size='24' fill='%230f766e'%3ESAF handle = permission that can survive reboot%3C/text%3E%3C/svg%3E)
The important distinction is whether the app owns the bytes or only has a user-granted document URI.

## Why a working path breaks later

On Android 10 and newer, the Storage Access Framework can return a `content://` document URI. A temporary copy or path may work during the current session, then fail after the process is killed or the phone reboots. Android requires `ContentResolver.takePersistableUriPermission()` for access that should survive those boundaries. The `file_picker` issue that led to this feature describes exactly this failure mode: a document library otherwise needs broad storage access or a second SAF package.

| Requirement | Correct contract | Recovery cost |
| --- | --- | --- |
| Upload once | `PlatformFile.readAsBytes()` or `readAsByteStream()` | Re-pick if the upload is abandoned |
| Reopen one document later | Persistable SAF file handle | Re-pick only after the user revokes access |
| Browse a user-selected folder | Persistable SAF directory grant | Store URI, not only a path |
| Scan the whole device | Not a normal `file_picker` use case | Play policy and storage review apply |

## Version-aware migration

The SAF feature first appeared in the `file_picker` 12.0.0 beta Android implementation as `AndroidSAFOptions`, `AndroidSAFHandle`, and `AndroidPlatformFile`. The old examples used an `androidSafOptions` argument. In the 13.x migration, that argument was removed from the top-level methods and the package directs callers to `androidOptions`; the federated Android package now owns the platform-specific implementation.

That means an upgrade can fail in two different ways: old code may stop compiling, or code may compile while the repository still stores only `path` and silently loses its durable access contract.

For a v12-era implementation, the intent looks like this. Pin the package versions before copying the symbol names because the 13.x surface was reshaped.

```dart
import 'package:file_picker/file_picker.dart';

Future<PlatformFile?> pickForLibrary() {
  return FilePicker.platform.pickFile(
    type: FileType.custom,
    allowedExtensions: ['pdf', 'docx'],
    androidSafOptions: const AndroidSAFOptions(
      grant: AndroidSAFGrant.lifetime,
      accessMode: AndroidSAFAccessMode.readOnly,
    ),
  );
}
```

The application layer should not serialize only `file.path`. Persist the SAF URI and access mode from the Android-specific result, keep a user-facing “Reconnect file” action, and treat a revoked grant as a recoverable state rather than a corrupted database row.

## A reproducible upgrade checklist

1. Pin the exact `file_picker` and Android implementation versions in `pubspec.lock`.
2. Pick a document from a provider such as Downloads or Drive, then force-stop the app.
3. Reboot the emulator or device and attempt to open the saved item.
4. Verify that the stored record contains a `content://` URI or SAF handle, not just a cache path.
5. Test user revocation, provider removal, and a read-only grant before shipping.
6. If the feature is a one-time upload, remove the persistent grant and copy the bytes into app-owned storage instead.

Do not fix this by adding `MANAGE_EXTERNAL_STORAGE`. Android's own guidance and the package's migration issue point to SAF for user-selected documents and folders; broad access adds Play review risk without fixing the ownership boundary.

The short rule is: a path tells Flutter where a file is now, while a persistable SAF grant records why Android should let the app access the user's selected resource later. Store the latter when the product promises a library, and keep the former for disposable uploads.

Sources checked 2026-10-17: [`file_picker` package API and migration notes](https://pub.dev/packages/file_picker), [`file_picker` changelog](https://pub.dev/packages/file_picker/changelog), [`android_file_picker` package](https://pub.dev/packages/android_file_picker), and the [SAF persistable-grant issue](https://github.com/vicajilau/flutter_file_picker/issues/1825).
