---
layout: post
title: "go_router 18 Migration - Restore Flutter Hero Animations After material_ui Split"
description: "go_router 18 can silently stop Hero flights in ShellRoute apps that still use Flutter's MaterialApp. Reproduce the regression, choose a safe migration path, and verify the fix."
date: 2026-10-21
tags: [go_router, navigation, Flutter, animation, migration, Web]
comments: true
share: true
---
![go_router nested navigation and Flutter Hero animation](/assets/images/go-router-stateful-shell-route.png)

The risky part of a go_router 18 upgrade is not always a compiler error. If an existing app still imports `package:flutter/material.dart`, a `ShellRoute` or `StatefulShellRoute` can keep rendering while its `Hero` transition quietly stops. Apps already using the standalone `material_ui` and `cupertino_ui` packages should migrate forward. Apps with a large legacy widget tree should pin go_router 17.5.0 first, then migrate the UI boundary deliberately.

## What changed in go_router 18

The 18.0.0 release moved its Material and Cupertino helpers to the standalone `material_ui` and `cupertino_ui` packages. The package also raised the minimum environment to Flutter 3.44 and Dart 3.12. The important compatibility detail is that `material_ui.MaterialApp` and Flutter's old `MaterialApp` are different Dart types, even though their names and roles look identical.

| App state | Recommended action | Why |
| --- | --- | --- |
| Flutter 3.44+, app already imports `material_ui` | Upgrade to go_router 18.x | The router and app use the same UI type family |
| Flutter 3.44+, app still imports `flutter/material.dart` everywhere | Stage an import and dependency migration | A clean build does not prove nested Hero behavior |
| Older Flutter or Dart SDK | Stay on go_router 17.5.x | go_router 18 cannot satisfy the SDK constraint |
| Release blocked by a visual regression | Pin 17.5.0 temporarily | It is a reversible recovery step, not the final migration |

## Reproduce the failure before changing production code

Use a small app with one normal route, one `ShellRoute`, and one `StatefulShellRoute`. Put the same `Hero` tag on a grid tile and its detail image. Keep the app imports on `package:flutter/material.dart`, then change only the package version.

```yaml
dependencies:
  flutter:
    sdk: flutter
  go_router: 17.5.0 # baseline
```

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

final router = GoRouter(
  routes: [
    ShellRoute(
      builder: (context, state, child) => AppShell(child: child),
      routes: [
        GoRoute(
          path: '/detail',
          builder: (context, state) => const DetailPage(),
        ),
      ],
    ),
  ],
);

void main() => runApp(
      MaterialApp.router(routerConfig: router),
    );
```

Run the same example with `go_router: 18.0.0`; validate the exact newer 18.x version you plan to ship separately. The reported failure mode is a snap instead of a Hero flight inside shell navigators; normal root navigation may still look fine. That difference is why a smoke test that checks only whether a route opens can miss the regression.

## Choose one migration boundary

The clean path is to move the app-level Material imports and dependencies together. The official `material_ui` migration guide provides a Dart fix for the import conversion:

```bash
flutter pub add material_ui cupertino_ui
dart fix --apply --code=migrate_design_widgets
flutter clean
flutter pub get
```

After the fix, the app entry point should use the standalone library:

```dart
import 'package:material_ui/material_ui.dart';
import 'package:go_router/go_router.dart';

void main() {
  runApp(MaterialApp.router(routerConfig: router));
}
```

Do not convert only the router file. A package or feature subtree that still expects Flutter's original `ThemeData`, `MaterialLocalizations`, or `MaterialApp` can form a split widget ecosystem. The compatibility bridge documented by `material_ui` is useful for a legacy subtree, but it should be treated as a migration seam with a test, not as proof that every third-party package is compatible.

## Verification checklist

Run this matrix on the exact Flutter channel used by CI:

| Check | Expected result |
| --- | --- |
| `flutter pub get` | No SDK or transitive dependency conflict |
| Cold start at a deep link | The intended shell branch opens |
| Tap grid item → detail | Hero flies, not just a route replacement |
| Switch shell branches and repeat | The animation still works after nested navigation |
| Android back and browser back | Route stack and URL remain correct |
| Release build | No legacy Material import is left in migrated boundaries |

If the app cannot migrate all dependencies in the release window, restore the known-good constraint and record the visual test as a release gate:

```yaml
go_router: 17.5.0
```

That rollback does not fix the compatibility problem. It prevents a silent UX regression while the import graph, localization delegates, themes, and third-party widgets are migrated together.

The key decision is therefore not “does go_router 18 compile?” It is “does the router see the same Material/Cupertino app types as the widget tree?” Check that boundary with a nested Hero reproduction before shipping.

References: [go_router 18 migration guide](https://flutter.dev/go/go-router-v18-breaking-changes), [go_router changelog](https://github.com/flutter/packages/blob/main/packages/go_router/CHANGELOG.md), [material_ui migration instructions](https://pub.dev/packages/material_ui), and [the reproducible Hero regression report](https://github.com/flutter/flutter/issues/192043).
