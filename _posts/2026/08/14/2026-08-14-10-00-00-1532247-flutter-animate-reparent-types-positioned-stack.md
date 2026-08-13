---
layout: post
title: "flutter_animate reparentTypes - Animate Flutter Positioned Widgets Inside a Stack"
description: "Learn how flutter_animate reparentTypes fixes parent constraints for Positioned widgets, with a working Stack example and practical animation caveats."
date: 2026-08-14
tags: [flutter_animate, animation, performance]
comments: true
share: true
---

![flutter_animate reparentTypes keeping a Positioned card inside a Flutter Stack](https://images.unsplash.com/photo-1558655146-d09347e92766?w=1200&q=80)

`flutter_animate` normally wraps a widget with an animation widget. That is convenient until the child is `Positioned`. A `Positioned` widget only works as an immediate child of `Stack`, so an animation wrapper can trigger the familiar “Incorrect use of ParentDataWidget” error. `Animate.reparentTypes` is the escape hatch: it tells `flutter_animate` how to keep the required parent relationship intact.

## The failure that looks unrelated to animation

This code looks reasonable, but the wrapper inserted by `.animate()` changes the tree around `Positioned`:

```dart
Stack(
  children: [
    Positioned(
      right: 16,
      bottom: 16,
      child: const Icon(Icons.add),
    ).animate().scale(),
  ],
)
```

`Stack` no longer sees `Positioned` as its direct child. Flutter then tries to apply `StackParentData` to the wrong render object. The stack trace points at layout, but the real cause is the extra animation parent.

## Register the parent-aware wrapper

The package exposes a static `reparentTypes` map. The key is the widget type that needs a special parent, and the value rebuilds that parent around the animated child. Register it once before the app starts:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

void configureFlutterAnimate() {
  Animate.reparentTypes[Positioned] = (parent, child) {
    final positioned = parent as Positioned;

    return Positioned(
      key: positioned.key,
      left: positioned.left,
      top: positioned.top,
      right: positioned.right,
      bottom: positioned.bottom,
      width: positioned.width,
      height: positioned.height,
      child: child,
    );
  };
}

void main() {
  configureFlutterAnimate();
  runApp(const MyApp());
}
```

Now the same animation can stay in the `Stack`:

```dart
Stack(
  children: [
    Positioned(
      right: 16,
      bottom: 16,
      child: const FloatingActionButton(
        onPressed: null,
        child: Icon(Icons.add),
      ),
    ).animate().fadeIn(duration: 220.ms).scale(begin: 0.8),
  ],
)
```

The important detail is that the callback must preserve every layout property. Copying only `right` and `bottom` works for one demo, then breaks when the same component receives `left`, `width`, or a custom `key`.

| Situation | Better choice | Reason |
|---|---|---|
| One fixed button in a `Stack` | Animate the inner child | No global registration needed |
| Reusable animated `Positioned` component | `reparentTypes` | Keeps layout and animation APIs together |
| Widget moves because layout changes | `AnimatedPositioned` | The position itself should drive layout |
| Widget only moves visually | `MoveEffect` | Siblings keep their original layout |

## A safer reusable helper

Global registration is useful, but it is also global mutable configuration. If a large app has several teams, hide it behind a small setup function and test it once. Do not register a different callback from a widget's `build` method; hot reload and rebuilds can make the setup difficult to reason about.

There is another boundary to keep clear: `reparentTypes` fixes the widget tree. It does not change how `ScaleEffect`, `MoveEffect`, or `FadeEffect` paint the child. If neighboring widgets must react to the new position, use `AnimatedPositioned` or an explicit layout animation instead of assuming `MoveEffect` changes layout.

`reparentTypes` is most valuable when a package-generated animation wrapper collides with Flutter's parent-data rules. Preserve the original parent properties, configure the map once, and choose a layout animation when the surrounding layout needs to move too.

- `Positioned` must remain an immediate child of `Stack`.
- `Animate.reparentTypes` rebuilds the required parent around the animated child.
- Copy all relevant `Positioned` fields, including `key`.
- Use `AnimatedPositioned` when the position is a layout concern.

- [flutter_animate Animate API](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/Animate-class.html)
- [Flutter Positioned documentation](https://api.flutter.dev/flutter/widgets/Positioned-class.html)
