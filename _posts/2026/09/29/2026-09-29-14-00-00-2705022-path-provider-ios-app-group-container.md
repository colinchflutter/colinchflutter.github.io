---
layout: post
title: "path_provider iOS App Group Container - Share Flutter Files with Extensions Safely"
description: "Fix Flutter iOS app extension file sharing by choosing the App Group container through path_provider_foundation, configuring both targets, and avoiding ordinary Documents paths."
date: 2026-09-29
tags: [path_provider, iOS, storage, app_extensions, Flutter]
comments: true
share: true
---

![Flutter iOS App Group container sharing files between the Runner app and an extension](/assets/images/shared-preferences-async-migration-cache-paths.png)

The right choice depends on who must read the file. A Flutter app that owns its files can use `getApplicationDocumentsDirectory()` or `getApplicationSupportDirectory()`. A Runner app and an iOS extension, such as a Share Extension or Widget, must use an App Group container instead. This does not apply to ordinary Android storage, and changing the Dart path alone cannot grant an extension access to the app's sandbox.

The failure is easy to misdiagnose: the file exists when the main app opens it, but the extension sees an empty directory or a permission error. The two targets have different sandboxes. `path_provider`'s normal directories are not a shared-storage contract.

## Choose the directory by ownership

| Data owner | API | Suitable data | Main trap |
| --- | --- | --- | --- |
| Runner only | `getApplicationDocumentsDirectory()` | User-created exports | An extension cannot assume this path |
| Runner only | `getApplicationSupportDirectory()` | Re-creatable app state and internal files | Still belongs to the app sandbox |
| Runner + iOS extension | `PathProviderFoundation.getContainerPath()` | Shared files or a SQLite database | Both targets need the same App Group entitlement |
| Temporary or rebuildable | `getTemporaryDirectory()` | Thumbnails, transient exports | The system may delete it |

Flutter's iOS extension guide names the same boundary: add the Runner and extension targets to one App Group, then use the App Group path from `path_provider` for files or `sqflite` for a database. The normal Documents directory is not a substitute.

## Configure Xcode before changing Dart

Use one exact identifier, for example `group.com.example.reader`, and apply it to both targets.

1. Open `ios/Runner.xcworkspace` in Xcode.
2. Select the Runner target, open **Signing & Capabilities**, and add **App Groups**.
3. Create or select `group.com.example.reader`.
4. Select the extension target and select the same group. Do not type a visually similar identifier by hand.
5. Confirm the Debug, Profile, and Release entitlements for both targets.
6. If the extension starts a Flutter engine, register the generated plugins for that target as described in Flutter's extension guide.

The App Group must also exist for the signing team in Apple's Developer account. A local build can look correctly configured while an archive fails when the provisioning profile does not contain the group.

## Resolve the shared path explicitly

`getContainerPath` is an API on the iOS/macOS implementation package, not a top-level function in `path_provider`. Add the implementation as a direct dependency when importing it directly.

```yaml
dependencies:
  path: ^1.9.1
  path_provider: ^2.1.6
  path_provider_foundation: ^2.6.0
```

Keep the App Group identifier in one configuration value. This example treats a missing container as a startup error instead of silently writing to a private directory.

```dart
import 'dart:io';

import 'package:path/path.dart' as p;
import 'package:path_provider_foundation/path_provider_foundation.dart';

const appGroupId = 'group.com.example.reader';

Future<File> sharedInboxFile(String name) async {
  final provider = PathProviderFoundation();
  final container = await provider.getContainerPath(
    appGroupIdentifier: appGroupId,
  );

  if (container == null || container.isEmpty) {
    throw StateError('App Group is unavailable: $appGroupId');
  }

  final directory = Directory(p.join(container, 'inbox'));
  await directory.create(recursive: true);
  return File(p.join(directory.path, name));
}
```

Call this only on iOS. The API is explicitly iOS-only and throws `UnsupportedError` on other platforms. A cross-platform repository should select an iOS App Group implementation behind a platform-specific adapter rather than sprinkling `Platform.isIOS` checks through feature code.

## The background and extension traps

`WidgetsFlutterBinding.ensureInitialized()` initializes Flutter bindings; it does not automatically make every plugin available to a separately created engine. A background callback or extension that reports `MissingPluginException` for `path_provider` needs plugin registration and an engine lifecycle check. Flutter has a confirmed issue showing the same symptom when `path_provider` is called from a Firebase Messaging background handler.

Test the storage contract as an upgrade scenario, not only as a clean install:

- Runner writes a file, then the extension reads it.
- The extension writes a file, then Runner reads it after returning to foreground.
- Both Debug and Release entitlements contain the same App Group.
- A missing group fails loudly rather than falling back to Documents.
- A file name is treated as untrusted input and stays inside the shared directory.
- SQLite users open the database under the App Group path and serialize concurrent writes.

The practical rule is short: use `path_provider`'s ordinary directories for one sandbox, and call `PathProviderFoundation.getContainerPath` for an iOS App Group. The entitlement and target configuration are part of the path; Dart code cannot repair a mismatch after the file has been created.

References: [Flutter iOS app extensions](https://docs.flutter.dev/platform-integration/ios/app-extensions), [path_provider_foundation API](https://pub.dev/documentation/path_provider_foundation/latest/path_provider_foundation/PathProviderFoundation-class.html), [path_provider_foundation example](https://pub.dev/packages/path_provider_foundation/example), and [Flutter issue #99445](https://github.com/flutter/flutter/issues/99445).
