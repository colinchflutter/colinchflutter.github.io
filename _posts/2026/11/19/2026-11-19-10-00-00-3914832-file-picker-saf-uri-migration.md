---
layout: post
title: "file_picker 12 Migration - Replace Android Paths with SAF URIs Without Breaking Folder Access"
description: "Migrating file_picker 12+ means replacing Android directory paths with content:// SAF URIs, updating file reads, and deciding when persistent access or a cache copy is safer."
date: 2026-11-19
tags: [file_picker, Android, migration, storage, performance]
comments: true
share: true
---

![Android Storage Access Framework document picker](https://images.unsplash.com/photo-1554224155-6726b3ff858f?w=1200&q=80)

If a Flutter app stores the result of `file_picker` as a filesystem path, the safest `file_picker` 12+ migration is to store a URI string and make the Android access decision explicit. This is useful for apps that let users choose a backup folder, import large documents, or reopen a document after a restart. It is not a drop-in fix for code that requires `dart:io File` everywhere; that code must either copy the selected content into app storage or add a URI-aware read boundary.

## The breaking boundary is the return type

Before the federated rewrite, an app could often treat a selected directory as a path. In current `file_picker`, `pickDirectory()` returns a `Uri?`. On Android, that URI normally starts with `content://` and represents a Storage Access Framework tree. On iOS, macOS, Windows, and Linux, it can be a `file` URI.

| Operation | Old assumption | Migration decision |
| --- | --- | --- |
| Choose one file | `FilePickerResult?` and `result.files` | `FilePicker.pickFile()` returns `PlatformFile?` |
| Choose several files | nullable wrapper | `FilePicker.pickFiles()` returns `List<PlatformFile>`; cancel means `[]` |
| Read bytes | `withData: true` | `await file.readAsBytes()` |
| Read as a stream | `withReadStream: true` | `file.readAsByteStream()` |
| Choose a directory | `String?` path | `Uri?`, and Android may return `content://` |
| Get a file length | `file.size` | `await file.length()` or `lengthSync() ?? await file.length()` |

The common failed migration is to call `selectedDirectory.toFilePath()` on every platform and save the result in a database. That works for a local `file://` URI, but a `content://` URI is not a path that `File()` can open. The resulting failure often appears later, after the settings screen has already closed, as a missing backup folder or `PathNotFoundException`.

## Keep the URI at the storage boundary

For a folder preference, persist the URI as text and reconstruct it only when the storage adapter needs it. Do not persist a guessed path beside it as the source of truth.

The following small repository keeps the platform difference visible instead of hiding it in a widget. The Android branch must use an SAF-capable implementation; the `file` branch can use `Directory.fromUri` or a normal path adapter in a desktop-only implementation.

```dart
import 'package:file_picker/file_picker.dart';

class FolderSelection {
  const FolderSelection(this.uri);

  final Uri uri;

  bool get isAndroidSaf => uri.scheme == 'content';
}

Future<FolderSelection?> chooseFolder() async {
  final uri = await FilePicker.pickDirectory();
  if (uri == null) return null;

  // Store uri.toString(), not uri.toFilePath().
  return FolderSelection(uri);
}

Future<List<int>> readPickedFile(PlatformFile file) async {
  // This works for the file_picker abstraction, including Android providers.
  return file.readAsBytes();
}
```

For large files, prefer `readAsByteStream()` and pass the stream to the upload or hashing layer. Copying a 1 GB video into a `Uint8List` turns a URI migration into a memory-pressure bug. Also treat `PlatformFile.lengthSync()` as an optimization, not a guaranteed value: the package documents that it can be `null` when the native picker did not provide a length.

## Choose between persistent access and copying

SAF is useful because the provider can represent local storage, an SD card, or a cloud document provider without exposing a raw path. The trade-off is lifecycle and permission state. The app should decide what it needs before saving the selection.

| Requirement | Better choice | Why |
| --- | --- | --- |
| Import once and process now | Read the URI, then copy into app storage | The app owns the copy and does not depend on a later provider permission |
| Reopen a user-selected folder after reboot | Use the package's Android SAF options and persist the grant handle/URI | A string alone is not a complete permission strategy |
| Upload a large file once | Stream from the picker abstraction | Avoids a second full-size cache copy |
| App-owned backup files | `path_provider` app documents/support directory | No user-selected URI is needed |
| Cross-device restore | Store a logical backup target, not a device URI | A URI from one device may not exist on another |

Android's Storage Access Framework grants access to the selected tree, not to every directory on the device. Android 11 and newer also restrict selection of locations such as `Android/data`, `Android/obb`, and some storage roots. The UI should report a rejected or unavailable location as a user choice, not as a generic “permission denied” crash.

## A migration sequence that can be reproduced

1. Update the dependency and run `flutter pub get` with the same Flutter/Dart minimum versions required by the selected `file_picker` release.
2. Replace `FilePickerResult?` handling with direct `PlatformFile?` or `List<PlatformFile>` handling. An empty list is cancellation for multi-file selection.
3. Replace `size`, `withData`, and `withReadStream` usage with the current methods.
4. Replace directory-path persistence with a serialized `Uri` and a provider-aware adapter.
5. Test a local Android folder, a cloud provider folder if installed, a denied selection, process restart, and a revoked or moved document.
6. Verify that the fallback copy is atomic: write to a temporary app-owned file, verify its length or hash, then rename it as the active backup.

For a minimal Android regression test, enable Developer options → Don't keep activities, select a document, background the app during the picker flow, and reopen the screen. The test is checking two separate things: whether the picker result can be recovered, and whether the app's persisted URI or copied file remains usable. Passing the first does not prove the second.

## Practical checklist

- [ ] No Android `content://` value is passed to `File()` or `Uri.toFilePath()`.
- [ ] Cancellation is handled as `null` for a directory and `[]` for multi-file selection.
- [ ] URI strings are treated as opaque identifiers, not edited with string path operations.
- [ ] Long-lived access uses the package's SAF option/handle API and is revalidated on startup.
- [ ] Large files use a stream or an app-owned copy, not an unconditional `readAsBytes()`.
- [ ] Android 11+ restricted folders are explained in the UI.
- [ ] A device restore or provider removal has a reselect path.

The durable fix is not a type cast from `Uri` to `String`. It is choosing whether the app owns the bytes or the user owns the document location, then keeping that decision in one storage adapter. Apps that only import a file once can avoid persistent SAF complexity by copying immediately. Apps that promise a reusable folder must retain and validate the provider permission instead of pretending that a URI is a normal filesystem path.

Sources checked on 2026-11-19: [file_picker package documentation](https://pub.dev/packages/file_picker), [file_picker changelog](https://pub.dev/packages/file_picker/changelog), [Android Storage Access Framework guide](https://developer.android.com/guide/topics/providers/document-provider), and [Android Intent URI permission reference](https://developer.android.com/reference/android/content/Intent).
