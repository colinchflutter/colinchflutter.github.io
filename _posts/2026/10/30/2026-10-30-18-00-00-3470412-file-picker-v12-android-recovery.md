---
layout: post
title: "file_picker v12 Migration - Recover Flutter Files After Android Activity Death"
description: "Migrate Flutter file_picker v11 code to v12/v13 and recover Android file selections after low-memory Activity destruction without treating a canceled pick as data loss."
date: 2026-10-30
tags: [file_picker, migration, Android, file_upload, recovery, testing]
comments: true
share: true
---

![Flutter file_picker migration and Android file recovery flow](https://images.unsplash.com/photo-1450101499163-c8848c66ca85?auto=format&fit=crop&w=1600&q=80)

`file_picker` v12 migration is not just a null-check rewrite. For an existing Flutter upload flow, change the result type, move byte loading to `PlatformFile`, and add Android lost-pick recovery before declaring the draft empty. This suits apps that must survive an upgrade while users select PDFs, images, or large documents. It does not make a temporary picker copy durable, and it does not replace server-side upload state.

## What changes between v11 and v12/v13

The package moved to a federated architecture in v12. The latest v13 line also makes `PlatformFile.length()` nullable, so code that assumes every provider reports a size needs one more boundary check.

| Existing code | Migration target | Production trap |
| --- | --- | --- |
| `FilePickerResult?` | `List<PlatformFile>` or `PlatformFile?` | Cancel now returns an empty list for `pickFiles()` |
| `allowMultiple: false` | `FilePicker.pickFile()` | `pickFiles()` defaults to multiple selection |
| `withData: true` | `file.readAsBytes()` | Large files no longer belong in the initial picker call |
| `withReadStream: true` | `file.readAsByteStream()` | The stream must be consumed and closed by the upload layer |
| `file.size` | `file.lengthSync() ?? await file.length()` | v13 can return `null`, which is not the same as zero |

For a single-document upload, make the new contract explicit:

```dart
import 'package:file_picker/file_picker.dart';

Future<PlatformFile?> pickDocument() async {
  final file = await FilePicker.pickFile(
    type: FileType.custom,
    allowedExtensions: <String>['pdf', 'docx'],
  );

  if (file == null) {
    return null; // The user canceled. Do not show a recovery error.
  }

  final bytesKnownWithoutIo = file.lengthSync();
  final length = bytesKnownWithoutIo ?? await file.length();
  if (length == null || length > 25 * 1024 * 1024) {
    throw const FormatException('The document size is unavailable or too large.');
  }

  return file;
}
```

The size check is deliberately separate from reading the payload. For a large upload, prefer `readAsByteStream()` and let the HTTP client stream it rather than creating another large `Uint8List`. If the backend requires a checksum or retryable body, persist the upload job and copy the selected content into app-owned storage before starting a long operation.

## Recover a selection after Android kills the Activity

Android can destroy the host Activity while its document picker is open. The original Dart `Future` then never completes, even though the user may have finished selecting a file. `file_picker` keeps the most recent lost result for `retrieveLostData()`.

Run recovery early in the restored upload screen. The platform check matters because `FilePickerAndroid` is an Android implementation, not a portable API:

```dart
import 'package:android_file_picker/android_file_picker.dart';
import 'package:file_picker_platform_interface/file_picker_platform_interface.dart';

Future<List<PlatformFile>> recoverLostFiles() async {
  if (FilePickerPlatform.instance case final FilePickerAndroid android) {
    final lost = await android.retrieveLostData();
    if (lost == null) {
      return <PlatformFile>[];
    }
    if (lost.exception != null) {
      throw lost.exception!;
    }
    return lost.files ?? <PlatformFile>[];
  }
  return <PlatformFile>[];
}
```

Treat the returned files like a normal successful pick, but show a different message such as “Your previous selection was restored.” The package documents that the result is delivered once and only the most recent lost pick is retained. Calling recovery only after the user taps the picker again is too late: a new request can replace the pending handoff.

To reproduce this path, enable Android developer option “Don’t keep activities,” open the picker, select a file, and return to the app. Also test a real low-memory restart; the developer option is a useful deterministic trigger, not proof that every device behaves identically.

## Do not confuse recovery with durable access

| Requirement | Correct boundary |
| --- | --- |
| Restore the result of a picker that just lost its Activity | `retrieveLostData()` during the restored flow |
| Upload a selected file after several minutes | Copy or stream it while the granted access is valid; keep an upload job separately |
| Reopen the same external Android document after a reboot | Use the package's current SAF/persistable-URI support and store the returned document identity, not only `PlatformFile.path` |
| Resume an interrupted upload | Server-side chunk/session ID plus local job metadata |

A path-looking string is not a universal durable identifier. Android document providers can return `content://` URIs, and a cache file can disappear when the system cleans temporary storage. The selected file should therefore be promoted to app-owned storage when the product promises offline retry or draft recovery.

## Upgrade checklist

- Replace `FilePickerResult` and `.files` with the v12 return types.
- Replace `size`, `withData`, and `withReadStream` usage with the new `PlatformFile` methods.
- Handle `length() == null` independently from an empty file.
- Call Android lost-pick recovery before rendering an empty upload draft.
- Keep cancellation, picker processing failure, and restored success as separate UI states.
- Test upgrade from an installed v11 build, Android Activity destruction, force-stop during upload, reboot, and a provider such as Google Drive.
- Do not claim a file is recoverable after reboot unless its URI or app-owned copy has been persisted and reopened successfully.

The practical migration is a boundary change: `file_picker` chooses a file, the upload repository owns bytes and retry state, and Android recovery only reconnects a lost picker result. That separation prevents a package upgrade from silently turning a valid user selection into an empty form.

References: [file_picker migration guide](https://pub.dev/packages/file_picker), [file_picker API documentation](https://pub.dev/documentation/file_picker/latest/), and [android_file_picker documentation](https://pub.dev/packages/android_file_picker) (checked 2026-10-30).
