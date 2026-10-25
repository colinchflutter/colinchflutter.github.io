---
layout: post
title: "path_provider Temporary Directory - Keep Flutter Upload Drafts from Disappearing"
description: "Choose the right path_provider directory for Flutter upload drafts, caches, and recovery files so Android and iOS cleanup does not erase unfinished work."
date: 2026-10-26
tags: [path_provider, caching, Android, iOS, migration]
comments: true
share: true
---

![Flutter path_provider temporary and persistent storage choices](https://images.unsplash.com/photo-1515879218367-8466d910aaa4?auto=format&fit=crop&w=1600&q=80)

The safest rule is simple: keep anything the user must recover in `getApplicationSupportDirectory()` or `getApplicationDocumentsDirectory()`, and reserve `getTemporaryDirectory()` for files that can be recreated. This applies to upload drafts, exported reports, and queued work; it does not mean every downloaded image belongs in Documents. A cache that disappears is normal. A draft that disappears is a data-loss bug.

## The failure that looks like an upload bug

An app copies a picked file into the temporary directory, stores only that path in a draft record, and resumes the draft later. The first run works. After the app is backgrounded, the device is low on storage, or the OS performs cleanup, the path still exists in the database but the file does not. The uploader then reports “file not found,” even though the user never deleted the draft.

Flutter’s `path_provider` documentation describes the temporary directory as purgeable at any time. The application documents directory survives until the app is deleted, while application support is intended for private support files. The important distinction is the recovery contract, not the directory name.

| Data | Recommended location | If missing, can the app recreate it? |
|---|---|---|
| Thumbnail or HTTP response cache | `getTemporaryDirectory()` | Yes |
| Resumable upload source or draft attachment | `getApplicationSupportDirectory()` | Usually no |
| User-exported file the app owns | `getApplicationDocumentsDirectory()` | No |
| Database containing draft metadata | Database-specific application data path | No |

The image above is a reminder to separate source files from disposable build artifacts; a path is not a guarantee that the bytes still exist.

## Copy the source, then persist the reference

Do not save the `XFile.path` returned by another plugin as if it were your durable storage. Copy the bytes into an application-owned directory before writing the draft record.

```dart
import 'dart:io';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';

Future<File> persistDraftAttachment(File source, String draftId) async {
  final root = await getApplicationSupportDirectory();
  final directory = Directory(p.join(root.path, 'drafts', draftId));
  await directory.create(recursive: true);

  final extension = p.extension(source.path);
  final target = File(p.join(directory.path, 'attachment$extension'));
  return source.copy(target.path);
}
```

Write the returned path only after `copy()` succeeds. On startup, treat a missing attachment as a recoverable draft state, not as a valid upload queue item. Show the draft title and a “choose file again” action, then remove only the broken attachment reference after the user confirms.

## A small recovery checklist

- Classify each file as disposable, recoverable, or user-exported.
- Store a relative draft path and a file size/hash, not only an absolute path.
- Re-check `File.exists()` before upload and after app restart.
- Delete support files only after the server confirms the upload.
- Test an interrupted copy, a missing file, and an app update separately.

Avoid moving everything to Documents as a blanket fix. Larger media may need an explicit cleanup policy, backup exclusion, or a user-visible export flow. The correct location is the one that matches how much data loss the product can tolerate.

Sources checked 2026-10-26: [Flutter file persistence recipe](https://docs.flutter.dev/cookbook/persistence/reading-writing-files), [path_provider package documentation](https://pub.dev/packages/path_provider), and [getApplicationSupportDirectory API](https://pub.dev/documentation/path_provider/latest/path_provider/getApplicationSupportDirectory.html).
