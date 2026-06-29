---
layout: post
title: "flutter_animate + ValueNotifier — Driving Animations from Business Logic Without StatefulWidget"
description: "How to use ValueNotifier and Animate.target to trigger flutter_animate effects from outside the widget tree, keeping animation state in your business logic layer without converting to StatefulWidget."
date: 2026-06-26
tags: [flutter_animate, animation, state_management, Flutter]
comments: true
share: true
---

![Flutter state management and animation coordination concept](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

Short answer: pass a `ValueNotifier<double>` to `Animate`'s `target` parameter. When the notifier changes, the animation plays. No `setState`, no `AnimationController`, no `StatefulWidget` conversion required.

Here's the thing that gets people — they reach for flutter_animate specifically to avoid boilerplate, then discover they still need a `StatefulWidget` to hold the toggle state. Turns out, you don't.

## Why This Comes Up

Say you have a card that should bounce when new data arrives. The new data comes from a repository or a notifier somewhere up the tree. The card itself is a `StatelessWidget`.

The naive path: convert the card to `StatefulWidget`, add a `bool _shouldBounce`, subscribe to the notifier, call `setState`. That's the exact boilerplate flutter_animate is supposed to eliminate.

The right path: hand the `ValueNotifier` directly to `Animate.target`.

## Basic Pattern: ValueNotifier<double> → target

```dart
class BounceCard extends StatelessWidget {
  const BounceCard({super.key, required this.bounceNotifier, required this.child});

  final ValueNotifier<double> bounceNotifier;
  final Widget child;

  @override
  Widget build(BuildContext context) {
    return Animate(
      target: bounceNotifier.value,
      effects: [
        ScaleEffect(
          begin: const Offset(1.0, 1.0),
          end: const Offset(1.04, 1.04),
          duration: 120.ms,
          curve: Curves.easeOut,
        ),
        ScaleEffect(
          begin: const Offset(1.04, 1.04),
          end: const Offset(1.0, 1.0),
          delay: 120.ms,
          duration: 150.ms,
          curve: Curves.elasticOut,
        ),
      ],
      child: child,
    );
  }
}
```

The catch: `Animate.target` only reads `bounceNotifier.value` once at build time. It won't listen for changes on its own. You still need something to rebuild the widget when the notifier fires. That's where `ValueListenableBuilder` comes in.

## Making It Actually Reactive

```dart
class BounceCard extends StatelessWidget {
  const BounceCard({super.key, required this.bounceNotifier, required this.child});

  final ValueNotifier<double> bounceNotifier;
  final Widget child;

  @override
  Widget build(BuildContext context) {
    return ValueListenableBuilder<double>(
      valueListenable: bounceNotifier,
      builder: (context, target, _) {
        return Animate(
          target: target,
          effects: [
            ScaleEffect(
              begin: const Offset(1.0, 1.0),
              end: const Offset(1.04, 1.04),
              duration: 120.ms,
            ),
            ScaleEffect(
              begin: const Offset(1.04, 1.04),
              end: const Offset(1.0, 1.0),
              delay: 120.ms,
              duration: 150.ms,
              curve: Curves.elasticOut,
            ),
          ],
          child: child,
        );
      },
    );
  }
}
```

`ValueListenableBuilder` rebuilds only the `Animate` widget — not the rest of the tree. When `bounceNotifier.value` changes from `0.0` to `1.0`, `Animate` sees a new `target` and plays the effects forward. Set it back to `0.0` to reverse.

Triggering from the outside:

```dart
// Somewhere in your repository layer or notifier:
void onNewMessageReceived() {
  bounceNotifier.value = bounceNotifier.value == 0.0 ? 1.0 : 0.0;
}
```

Flip-flopping between 0 and 1 is the reliable pattern. You can't just set it to 1 twice — `Animate` needs to see an actual value *change* to restart.

## Driving from a ChangeNotifier

If you're already using `ChangeNotifier` (or Riverpod/BLoC/anything with a stream), wrapping that in a `ValueNotifier` is overkill. Just use `AnimatedBuilder`:

```dart
class NotificationBadge extends StatelessWidget {
  const NotificationBadge({super.key, required this.notifier});

  final MessageNotifier notifier;

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: notifier,
      builder: (context, _) {
        return Animate(
          // target toggles each time message count increases
          target: notifier.unreadCount > 0 ? 1.0 : 0.0,
          effects: [
            ShakeEffect(
              duration: 400.ms,
              hz: 4,
              offset: const Offset(2, 0),
            ),
          ],
          child: Badge(
            count: notifier.unreadCount,
          ),
        );
      },
    );
  }
}
```

The `AnimatedBuilder` listens to `notifier` (which extends `ChangeNotifier`, so it's a `Listenable`). Every time `notifyListeners()` fires, the builder reruns, and if `unreadCount` flipped across zero, `Animate` gets a new target and plays the shake.

![Flutter animation driven by external state change notification](https://images.unsplash.com/photo-1614728894747-a83421e2b9c9?w=800&q=80)

## Pattern: Toggle Between Two States

For binary animations — open/closed, selected/unselected, expanded/collapsed — a `ValueNotifier<bool>` is cleaner than a double:

```dart
class ExpandableSection extends StatelessWidget {
  const ExpandableSection({super.key, required this.isExpanded, required this.child});

  final ValueNotifier<bool> isExpanded;
  final Widget child;

  @override
  Widget build(BuildContext context) {
    return ValueListenableBuilder<bool>(
      valueListenable: isExpanded,
      builder: (context, expanded, _) {
        return Animate(
          target: expanded ? 1.0 : 0.0,
          effects: [
            ExpandEffect(
              alignment: Alignment.topCenter,
              duration: 300.ms,
              curve: Curves.easeInOut,
            ),
            FadeEffect(duration: 250.ms),
          ],
          child: child,
        );
      },
    );
  }
}
```

Call site:

```dart
final isExpanded = ValueNotifier<bool>(false);

// Widget tree:
GestureDetector(
  onTap: () => isExpanded.value = !isExpanded.value,
  child: HeaderWidget(),
),
ExpandableSection(
  isExpanded: isExpanded,
  child: ContentWidget(),
),
```

Zero `StatefulWidget`. The `ValueNotifier` lives wherever makes sense — in the parent widget's field, in a provider, in a controller class.

## The Gotcha: Where Does the Notifier Live?

This is where people hit a wall. A `ValueNotifier` needs to be created *once* and disposed properly. If you create it inside a `StatelessWidget`'s `build` method, it gets recreated on every rebuild — animations won't work correctly and you'll leak the notifier.

Three valid options:

**1. Parent StatefulWidget holds it (simplest)**

```dart
class ParentWidget extends StatefulWidget {
  // ...
}

class _ParentWidgetState extends State<ParentWidget> {
  final _bounceNotifier = ValueNotifier<double>(0.0);

  @override
  void dispose() {
    _bounceNotifier.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return BounceCard(bounceNotifier: _bounceNotifier, child: ...);
  }
}
```

One `StatefulWidget` at the right level. The animation widgets below stay stateless.

**2. Provider / DI layer**

If you're using Riverpod or a service locator, create the notifier there and inject it. The notifier's lifecycle matches the feature, not a widget.

**3. `flutter_hooks` useValueNotifier()**

```dart
final bounceNotifier = useValueNotifier(0.0);
```

One line, proper lifecycle, no `StatefulWidget`. Works great if you're already using `flutter_hooks`.

## Performance Note

`ValueListenableBuilder` rebuilds its subtree on every notifier change. If your animation is deep inside a large widget tree, wrap tightly — only rebuild what needs the animation target, not the whole page.

```dart
// Don't wrap the entire Scaffold
// Do wrap just the animated widget
ValueListenableBuilder<double>(
  valueListenable: notifier,
  builder: (context, target, _) => AnimatedWidget(target: target),
)
```

---

Next up: profiling flutter_animate on low-end devices — how to catch dropped frames before they hit users, and which effects are cheapest vs. most expensive.

**References**
- [flutter_animate pub.dev](https://pub.dev/packages/flutter_animate)
- [ValueNotifier Flutter docs](https://api.flutter.dev/flutter/foundation/ValueNotifier-class.html)
- [flutter_animate GitHub](https://github.com/gskinner/flutter_animate)
