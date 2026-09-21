---
layout: post
title: "permission_handler iOS Flavors - Stop Debug Permissions Leaking into Production"
description: "Fix permission_handler iOS flavor builds that compile the wrong permissions by selecting the correct Info.plist, clearing stale package caches, and verifying the release configuration."
date: 2026-09-21
tags: [permission_handler, iOS, migration, debugging, permissions]
comments: true
share: true
---

![permission_handler iOS flavor permission configuration](https://pub.dev/static/img/pub-dev-icon.svg)

For a Flutter app with separate dev and production iOS flavors, use `permission_handler.yaml` to select one `Info.plist` per build configuration. This is safer than merging every plist and hoping the Pod or Swift package sees the intended one. A single-flavor app does not need this extra manifest, and an Android-only permission issue will not be fixed by changing iOS settings.

## The failure is configuration-dependent

Suppose the development build requests camera and microphone access, while the production build only needs photos. If `Info-dev.plist` and `Info-prod.plist` are both visible through Xcode configurations, their keys can be merged. The production binary can then contain a permission it never uses, while a build started from Xcode may read no plist at all and report every permission as denied.

| Build situation | Likely result | Better decision |
|---|---|---|
| One app target, one plist | Keys are predictable | Keep the simple setup |
| Dev and prod need different permissions | Unwanted keys can be compiled into both builds | Select a flavor plist explicitly |
| Build launched from Xcode.app | Working directory detection can fail | Set `PERMISSION_HANDLER_INFO_PLIST` |
| Changed plist or permission macro | Cached package manifest may be reused | Clear DerivedData before rebuilding |

## Select one plist instead of merging them

Place this file beside `pubspec.yaml`. This is the flavor workflow documented by the [official permission_handler package guide](https://pub.dev/packages/permission_handler). The configuration names must match the Xcode build configurations actually used by the Flutter flavors.

```yaml
strict: true
flavors:
  dev:
    info-plist: ios/Runner/Info-dev.plist
    configurations: [Debug-dev, Profile-dev, Release-dev]
  prod:
    info-plist: ios/Runner/Info-prod.plist
    configurations: [Debug-prod, Profile-prod, Release-prod]
```

Keep only the usage descriptions needed by each build. For example, `Info-dev.plist` may contain `NSCameraUsageDescription`, while `Info-prod.plist` can omit it if production never calls `Permission.camera`. The Dart call still needs to match the selected native configuration; removing a key does not turn a denied permission into a usable one.

Select before building, then remove stale Xcode package output when changing the selection:

```bash
dart run permission_handler_apple:select prod
rm -rf "$HOME/Library/Developer/Xcode/DerivedData"
flutter build ios --release --flavor prod
```

The `rm` command targets Xcode's derived data only. If the build is launched from Xcode.app rather than the terminal, set the plist path explicitly for that session and restart Xcode:

```bash
launchctl setenv PERMISSION_HANDLER_INFO_PLIST \
  "$PWD/ios/Runner/Info-prod.plist"
```

## Verify the artifact before blaming Dart code

Use this checklist for a release candidate:

- [ ] The selected flavor is printed or recorded in the build step.
- [ ] The selected plist contains every usage description requested by Dart.
- [ ] The production plist does not contain development-only permission keys.
- [ ] DerivedData was cleared after changing plist paths or macros.
- [ ] The app was tested once from the same Xcode configuration used for archive.
- [ ] A command-line diagnostic confirms which plist was read when the result is `denied`.

For a missing or wrongly detected plist, the package documentation provides a verbose manifest check. Run it from the Flutter project root and inspect the resolved file and permission macros:

```bash
PERMISSION_HANDLER_VERBOSE=1 \
swift package --manifest-cache none \
  --package-path ios/Flutter/ephemeral/Packages/.packages/permission_handler_apple \
  dump-package > /dev/null
```

The practical boundary is simple: use one plist when every build needs the same permissions; use flavor selection when the native capability set differs. Do not “fix” a denied production permission by copying the development plist into the release target. That hides the configuration error and can ship an unnecessary permission declaration. For the reason each usage-description key is required, cross-check Apple's [requesting access to protected resources](https://developer.apple.com/documentation/bundleresources/information-property-list/protected_resources) documentation.
