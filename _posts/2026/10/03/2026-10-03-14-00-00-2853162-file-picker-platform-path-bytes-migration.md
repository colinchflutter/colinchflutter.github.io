---
layout: post
title: "file_picker v12/v13 Migration - Fix Flutter Web path Assumptions with bytes"
description: "Migrate file_picker v12/v13 safely by separating native paths from Web bytes, handling nullable length, and avoiding upload failures after upgrades."
date: 2026-10-03
tags: [file_picker, migration, networking, Web, Android, iOS]
comments: true
share: true
---

<svg xmlns="http://www.w3.org/2000/svg" width="100%" viewBox="0 0 900 190" role="img" aria-label="file_picker uses bytes on Web and an optional path on native platforms"><rect width="900" height="190" rx="18" fill="#172338"/><text x="32" y="42" fill="#f8fafc" font-family="Arial,sans-serif" font-size="25" font-weight="700">file_picker: choose a content contract</text><rect x="35" y="78" width="205" height="70" rx="12" fill="#1e293b"/><text x="68" y="121" fill="#bfdbfe" font-family="Arial,sans-serif" font-size="22">PlatformFile</text><path d="M250 113h95" stroke="#60a5fa" stroke-width="6"/><path d="M338 105l12 8-12 8" fill="none" stroke="#60a5fa" stroke-width="6"/><rect x="365" y="68" width="185" height="55" rx="12" fill="#0f766e"/><text x="425" y="103" fill="#ecfeff" font-family="Arial,sans-serif" font-size="21">Web → bytes</text><rect x="365" y="130" width="185" height="45" rx="12" fill="#7c3aed"/><text x="395" y="159" fill="#f5f3ff" font-family="Arial,sans-serif" font-size="18">Native → path?</text><path d="M560 95h90" stroke="#5eead4" stroke-width="6"/><path d="M643 87l12 8-12 8" fill="none" stroke="#5eead4" stroke-width="6"/><text x="675" y="104" fill="#f8fafc" font-family="Arial,sans-serif" font-size="21">upload</text></svg>

The safe migration rule for `file_picker` v12 or v13 is simple: treat `PlatformFile.path` as an optional native optimization, not as the file itself. Use `readAsBytes()` for a cross-platform upload, and use `readAsByteStream()` when the file may be large. This matters most to Flutter Web apps and shared upload code; a native-only app that already consumes a local path can keep that path, but it still needs null handling after the upgrade.

## The failure hidden by a native test

Older code often looks like this:

```dart
final result = await FilePicker.platform.pickFiles();
final path = result?.files.single.path;
await uploadFromLocalPath(path!);
```

It has two independent assumptions. A canceled picker returns no usable file, and Web does not provide a filesystem path that `dart:io` can open. A public issue reported a Web upgrade where `path` became a data/blob value rather than a local path. Logging it can produce a huge string, while passing it to a native file API fails outright.

The package documentation also changes the migration surface: v12 returns `List<PlatformFile>` directly, removes `FilePickerResult`, and replaces `withData`/`withReadStream` with methods on `PlatformFile`. v13 makes `length()` return `Future<int?>`, so an unknown length is no longer confused with an empty file.

## Choose the representation at the upload boundary

| Situation | Use | Avoid |
| --- | --- | --- |
| Web upload or shared API client | `await file.readAsBytes()` | `File(file.path!)` |
| Large upload with a streaming client | `file.readAsByteStream()` | Loading the whole file into RAM |
| Native-only image transform | `file.path` after a null check | Assuming the same path exists on Web |
| File size validation | `file.lengthSync() ?? await file.length()` | `file.size` from v11 |

This keeps platform branching near the code that truly needs a native path. The picker and upload layers can remain shared:

```dart
import 'dart:typed_data';
import 'package:file_picker/file_picker.dart';

class PickedUpload {
  const PickedUpload({
    required this.name,
    required this.bytes,
    required this.length,
  });

  final String name;
  final Uint8List bytes;
  final int length;
}

Future<PickedUpload?> pickUpload() async {
  final file = await FilePicker.pickFile();
  if (file == null) return null; // cancel is a normal result

  final bytes = await file.readAsBytes();
  final length = file.lengthSync() ?? await file.length();
  if (length == null) {
    throw StateError('The selected file length could not be read');
  }

  return PickedUpload(
    name: file.name,
    bytes: bytes,
    length: length,
  );
}
```

The code deliberately reads the content instead of converting `path` into a `File`. For a small document or image this is the least surprising contract. For a large file, change the upload API to accept `Stream<Uint8List>` and call `readAsByteStream()`; do not silently replace a stream with `readAsBytes()` just to preserve an old method signature.

## Migration checklist

1. Replace `FilePickerResult?` and `result.files` with `List<PlatformFile>` or use `FilePicker.pickFile()` for one file. Keep the empty list/null branch as the cancellation path.
2. Remove `withData`, `withReadStream`, and `size` usage. Move content and size reads to `readAsBytes()`, `readAsByteStream()`, and `length()`.
3. Search for `path!`, `File(`, `MultipartFile.fromFile`, and image APIs that receive a path. Those call sites need a native-only branch or a bytes-based alternative.
4. Test at least one Web upload and one Android/iOS upload. A native emulator test cannot prove that Web path assumptions are safe.
5. Pin the package only as a short rollback measure. The durable fix is to make the upload boundary accept bytes or a stream, then remove the pin after the migrated path is covered.

The package's compatibility table is also a useful guardrail: `pickFile()` and `pickFiles()` work across the listed platforms, while directory APIs do not exist on Web. A feature that needs a directory path is therefore a different product requirement from selecting a file for upload.

The verified references are the [`file_picker` migration notes](https://pub.dev/packages/file_picker), its [changelog](https://pub.dev/packages/file_picker/changelog), and the [Web `path` issue](https://github.com/miguelpruivo/flutter_file_picker/issues/1998). The issue is evidence of a failure mode, not a promise that every version returns the same value; the stable design decision is still to avoid using a filesystem path on Web.

The short version: `PlatformFile` is the selection result, `bytes` or a stream is the portable content contract, and `path` is an optional native detail. Make that distinction before upgrading, and the v12/v13 migration stops being a platform-specific upload incident.
