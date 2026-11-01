---
layout: post
title: "share_plus iPad Share Sheet - Fix Anchor Crashes and Temporary File Leaks in Flutter"
description: "Use share_plus safely on iPad and across repeated file shares by supplying a real popover anchor and controlling temporary XFile lifetime."
date: 2026-11-02
tags: [share_plus, iOS, file_sharing, Flutter, performance]
comments: true
share: true
---

![Flutter share_plus iPad popover anchor and temporary file lifecycle](https://images.unsplash.com/photo-1556656793-08538906a9f8?auto=format&fit=crop&w=1600&q=80)

If a Flutter app shares files from both iPhone and iPad, use `share_plus` with two explicit rules: pass the tapped widget's rectangle as `sharePositionOrigin` on iPad, and treat `XFile.fromData` output as cache data that needs cleanup. The iPad fix is not a general layout decoration, and the cleanup rule is unnecessary when you already share a file with a stable path.

## Two failures that look unrelated

`share_plus` delegates to the platform share UI. On iPad, that UI is presented as a popover. Without an origin rectangle, the package documentation says the sheet can be anchored at the center; older platform combinations could also leave the UI unresponsive or crash. A test on an iPhone does not exercise this path.

The second trap appears when a report, image, or CSV exists only in memory. `XFile.fromData` lets the package create a temporary file in the app cache so another app can read it. That is convenient, but repeated exports can leave files until the operating system decides to reclaim them. Cache cleanup is not a guarantee from the OS, so a long-lived app should own a cleanup policy.

| Sharing case | Safer choice | Why |
|---|---|---|
| Text or URL from a button | `ShareParams(text: ..., uri: Uri.parse(...))` | No file lifecycle to manage |
| Existing app-owned file | `XFile(path)` | The path and retention policy are explicit |
| Bytes generated in memory | Write to cache, then share `XFile(path)` | Cleanup can be scheduled by the app |
| iPad share sheet | Always provide `sharePositionOrigin` | The popover needs a source rectangle |

## Anchor the share sheet to the actual control

The rectangle should come from the render box that the user tapped. A constant rectangle may hide the problem in a simulator but can point outside the visible window after rotation, split view, or a different tablet layout.

```dart
import 'package:flutter/material.dart';
import 'package:share_plus/share_plus.dart';

class ShareButton extends StatelessWidget {
  const ShareButton({required this.text, super.key});

  final String text;

  Future<void> _share(BuildContext context) async {
    final renderObject = context.findRenderObject();
    final box = renderObject is RenderBox ? renderObject : null;
    if (box == null || !box.hasSize) return;

    final origin = box.localToGlobal(Offset.zero) & box.size;
    await SharePlus.instance.share(
      ShareParams(
        text: text,
        sharePositionOrigin: origin,
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Builder(
      builder: (buttonContext) => IconButton(
        icon: const Icon(Icons.share),
        onPressed: () => _share(buttonContext),
      ),
    );
  }
}
```

`Builder` gives the callback a context below the button, so the render object belongs to the control rather than a larger page container. Keep the origin calculation in the action that opens the sheet; storing it during `build` makes it stale after rotation or a layout change.

## Give generated files a bounded lifetime

For a report generated as bytes, create an app-owned cache file with a recognizable prefix. The share action can then use the ordinary path-based `XFile` API, while startup removes files older than a conservative retention window.

```dart
import 'dart:io';
import 'package:path_provider/path_provider.dart';
import 'package:share_plus/share_plus.dart';

Future<void> shareReport(List<int> bytes) async {
  final directory = await getTemporaryDirectory();
  final file = File(
    '${directory.path}/share_report_${DateTime.now().microsecondsSinceEpoch}.pdf',
  );
  await file.writeAsBytes(bytes, flush: true);

  await SharePlus.instance.share(
    ShareParams(files: [XFile(file.path)]),
  );
}

Future<void> cleanOldShareReports({
  Duration maxAge = const Duration(days: 2),
}) async {
  final directory = await getTemporaryDirectory();
  final cutoff = DateTime.now().subtract(maxAge);

  await for (final entity in directory.list()) {
    if (entity is! File || !entity.path.contains('/share_report_')) {
      continue;
    }
    final modified = await entity.lastModified();
    if (modified.isBefore(cutoff)) {
      await entity.delete();
    }
  }
}
```

Call `cleanOldShareReports()` after Flutter bindings and path-provider initialization, not while the user is waiting for the share sheet. The exact two-day window is an application choice, not a promise from `share_plus`; choose a longer period if a receiving app may read the file after the share call returns. Do not delete the file immediately in a `finally` block unless the platform and receiving apps have been tested with that timing.

If you use `XFile.fromData`, keep the same startup cleanup idea, but use the package's documented cache location and filename pattern for the version you ship. A path you create yourself is easier to audit and test. Never delete every file in the temporary directory: other plugins own that directory too.

## Release checklist

- Test the share action on iPhone and iPad, including rotation and split view.
- Verify the origin rectangle comes from the tapped control, not `MediaQuery` dimensions.
- Share a large generated file, background the app, and open it in at least two receiver apps.
- Relaunch after several exports and confirm old `share_report_` files are removed.
- Keep user documents outside the temporary directory when the product promises later access.
- Treat a canceled share as a normal result; it is not a file-write failure.

The practical boundary is simple: `share_plus` owns presentation, while the app owns the anchor and any generated file's lifetime. That division prevents an iPad-only UI failure from being diagnosed as a Flutter layout bug and keeps export caches from becoming an invisible storage feature.

Sources: [share_plus package documentation](https://pub.dev/packages/share_plus), [share_plus changelog](https://pub.dev/packages/share_plus/changelog), and [path_provider package](https://pub.dev/packages/path_provider).
