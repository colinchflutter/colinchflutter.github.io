---
layout: post
title: "share_plus 13 Migration - Replace Deprecated Share APIs Without Breaking Flutter Builds"
description: "Migrate Flutter apps from deprecated share_plus static APIs to SharePlus.instance.share while checking the new Dart, Flutter, Android, iOS, and iPad constraints."
date: 2026-11-20
tags: [share_plus, migration, Flutter, Android, iOS, testing]
comments: true
share: true
---

![share_plus 13 Flutter migration from static APIs to ShareParams](https://images.unsplash.com/photo-1558655146-d09347e92766?auto=format&fit=crop&w=1600&q=80)

The safe `share_plus 13` migration is a two-part change: replace the deprecated static `Share.share*` calls with `SharePlus.instance.share(ShareParams(...))`, then verify the project toolchain before resolving dependencies. This upgrade fits an app already on Dart 3.11 and the current Flutter Android/iOS build stack. It does not fit an app that must keep an older Flutter SDK, Android Gradle Plugin, or iOS deployment target unchanged.

## Why a small API change can fail at release time

The old code looks harmless:

```dart
await Share.share(message);
await Share.shareUri(Uri.parse(url));
await Share.shareXFiles(files);
```

The current package keeps these static methods only for compatibility and marks them deprecated. The replacement moves the payload into one `ShareParams` object. That part is mechanical. The less obvious part is that `share_plus 13.0.0` also raises the project floor: Flutter 3.41.6, Dart 3.11, iOS 13, macOS 10.15, Android Gradle Plugin 8.12.1, Gradle 8.13, and Kotlin 2.2.0.

| Check | If it passes | If it fails |
| --- | --- | --- |
| Dart / Flutter SDK | Migrate the Dart calls now | Pin `share_plus` and schedule the SDK upgrade |
| Android build tools | Resolve 13 without changing the native build | Upgrade AGP, Gradle, and Kotlin together |
| iOS deployment target | Keep the 13.x package | Raise the target or stay on the previous major |
| iPad share origin | Pass the tapped widget rectangle | The popover can appear in the wrong place or become fragile |
| Share result handling | Preserve success/cancel/error behavior | Re-check UI messages and analytics |

This is a migration decision table, not a recommendation to raise every platform minimum blindly. If the application still supports iOS 12 or an older Android build pipeline, the free option is to remain on the compatible major until the platform migration is planned.

## Convert the call site without changing the product flow

Keep the existing text, URI, files, and result handling. Change the package boundary first so a review can distinguish API migration from behavior changes.

```dart
import 'package:share_plus/share_plus.dart';

Future<ShareResult> shareInvite({
  required String message,
  Uri? inviteUri,
}) {
  return SharePlus.instance.share(
    ShareParams(
      text: message,
      uri: inviteUri,
      title: 'Invite',
    ),
  );
}
```

For files, use the same object instead of a separate static method:

```dart
final result = await SharePlus.instance.share(
  ShareParams(
    text: 'Build report',
    files: files,
    subject: 'Flutter build report',
  ),
);

if (result.status == ShareResultStatus.success) {
  analytics.logEvent(name: 'share_completed');
}
```

The migration is not complete if the old call is hidden in a helper, a platform-specific branch, or a test fixture. Search the whole Dart source, not just the button widget:

```bash
rg "Share\.share|Share\.shareUri|Share\.shareXFiles" lib test
```

Then replace each occurrence with a named helper where possible. A single helper also gives the app one place to add analytics and to keep cancelled shares separate from failures.

## Keep iPad geometry in the new API

`ShareParams` is more than a renamed argument bag on iPad. The native share UI is a popover, so it needs a source rectangle. Obtain that rectangle from the button that initiated the action.

```dart
Builder(
  builder: (context) {
    return IconButton(
      icon: const Icon(Icons.share),
      onPressed: () async {
        final box = context.findRenderObject() as RenderBox?;
        if (box == null) return;

        await SharePlus.instance.share(
          ShareParams(
            text: 'A document from the app',
            sharePositionOrigin:
                box.localToGlobal(Offset.zero) & box.size,
          ),
        );
      },
    );
  },
)
```

Do not cache a rectangle from a previous build. A rotated device, a changed toolbar, or a modal route can make it stale. Calculate it from the current button context immediately before sharing. On Android and iPhone this value is usually not needed, but passing it does not remove the platform-specific iPad requirement.

## Upgrade the native toolchain as one unit

Run the dependency solver only after checking the native files. A partial upgrade often produces a Gradle or CocoaPods error that looks unrelated to `share_plus`.

```bash
flutter --version
dart --version
rg "compileSdk|com.android.tools.build:gradle|distributionUrl|kotlin" android
rg "IPHONEOS_DEPLOYMENT_TARGET|platform :ios" ios
```

If the project is below the package floor, choose one of these paths:

1. Stay on the existing `share_plus` major and keep the release branch stable.
2. Upgrade Flutter, Dart, AGP, Gradle, Kotlin, and the iOS deployment target in a dedicated branch, then move to 13.
3. Upgrade the package and toolchain together only when the app has a testable release candidate for both stores.

Do not solve a Kotlin or AGP mismatch by adding random dependency overrides. That can make one machine resolve while CI or a clean checkout fails. The build tools are part of this migration's compatibility contract.

## A short release checklist

- [ ] `rg` returns no deprecated static `Share.share*` calls in `lib` or `test`.
- [ ] Text, URI, and file shares each use the intended `ShareParams` field.
- [ ] iPad calculates `sharePositionOrigin` from the current share button.
- [ ] Cancelled shares are not recorded as failures.
- [ ] Android build succeeds from a clean checkout with the new Gradle/Kotlin floor.
- [ ] iOS builds with the selected deployment target and shows the share sheet on a real device.
- [ ] A file share is repeated after returning from the target app, so temporary-file cleanup is not accidentally coupled to API migration.

The key boundary is simple: the Dart API replacement is safe only when the project can satisfy the package's native build requirements. For an older app, pinning the previous major is a valid production choice. For a current app, move to `SharePlus.instance.share`, keep iPad geometry explicit, and test the result states before shipping.

Sources checked 2026-11-20: [share_plus package documentation](https://pub.dev/packages/share_plus), [share_plus changelog](https://pub.dev/packages/share_plus/changelog), and [Share Plus documentation indexed by Context7](https://pub.dev/packages/share_plus).
