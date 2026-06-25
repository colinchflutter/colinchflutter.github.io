---
layout: post
title: "flutter_animate target parameter — state-driven animations without AnimationController"
description: "Use flutter_animate's animate(target:) to toggle animations from state changes. Builds toggle effects, step-by-step reveals, and conditional transitions in a fraction of the code a vanilla AnimationController needs."
date: 2026-06-24
tags: [flutter_animate, animation, state_management, Flutter, Dart]
comments: true
share: true
---
![Toggle animation state change concept on mobile screen](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)

Most animations you write in Flutter fall into two categories: play once on mount (what the first few posts in this series covered), or toggle between two states based on something the user did. The second category is where Flutter's vanilla API gets painful fast. `flutter_animate` has a clean answer: the `target` parameter.

## What target does

`animate(target: value)` drives the animation to a specific position, where `0.0` is the beginning and `1.0` is the end. When `value` changes, the widget automatically animates to the new position.

Short answer: it's like `AnimatedOpacity`, except it works for any combination of effects you stack on `.animate()`.

```dart
bool _isVisible = false;

Widget build(BuildContext context) {
  return SomeWidget()
    .animate(target: _isVisible ? 1.0 : 0.0)
    .fadeIn(duration: 300.ms)
    .slideY(begin: 0.2, end: 0);
}
```

Flip `_isVisible`, call `setState`, done. The widget fades and slides in when `target` goes to 1.0, reverses out when it goes back to 0.0.

## The problem it replaces

Before I found `target`, a simple show/hide toggle looked like this:

```dart
class _PanelState extends State<Panel> with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  late final Animation<double> _fade;
  late final Animation<Offset> _slide;
  bool _open = false;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 300),
    );
    _fade = CurvedAnimation(parent: _ctrl, curve: Curves.easeIn);
    _slide = Tween<Offset>(begin: const Offset(0, 0.2), end: Offset.zero)
        .animate(CurvedAnimation(parent: _ctrl, curve: Curves.easeOut));
  }

  void toggle() {
    if (_open) {
      _ctrl.reverse();
    } else {
      _ctrl.forward();
    }
    setState(() => _open = !_open);
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return FadeTransition(
      opacity: _fade,
      child: SlideTransition(
        position: _slide,
        child: YourPanel(),
      ),
    );
  }
}
```

That's 35 lines. The `target` version is 6. I spent a morning refactoring four of these in one screen.

## Reversals are automatic

One thing that trips people up with the vanilla controller: you have to decide whether to call `.forward()` or `.reverse()`, which means tracking state in two places. With `target`, the direction is implicit — if the new target is higher than the current position, it goes forward; if lower, it reverses. The controller management is invisible.

![Flutter animation direction switching diagram](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

## Step-by-step reveals

`target` doesn't have to be just 0 or 1. You can pass any value between 0.0 and 1.0 to stop partway through the animation sequence.

```dart
int _step = 0; // 0, 1, 2, 3

Widget build(BuildContext context) {
  return Column(
    children: [
      StepOne()
        .animate(target: _step >= 1 ? 1.0 : 0.0)
        .fadeIn(duration: 250.ms),

      StepTwo()
        .animate(target: _step >= 2 ? 1.0 : 0.0)
        .fadeIn(duration: 250.ms),

      StepThree()
        .animate(target: _step >= 3 ? 1.0 : 0.0)
        .fadeIn(duration: 250.ms),
    ],
  );
}
```

Each step appears when `_step` crosses its threshold. Tap a button, increment `_step`, and the next item fades in. No coordinating multiple controllers or `Interval` curves.

## Combining target with onComplete callbacks

`animate(target:)` fires its `onComplete` callback when the animation reaches the target. Useful for chaining actions after the animation settles:

```dart
SomeWidget()
  .animate(
    target: _isOpen ? 1.0 : 0.0,
    onComplete: (controller) {
      if (!_isOpen) {
        // animation finished closing — now remove from widget tree
        setState(() => _shouldRender = false);
      }
    },
  )
  .fadeIn(duration: 300.ms)
  .scaleXY(begin: 0.95, end: 1.0, duration: 300.ms);
```

This is the pattern for animating-out before unmounting. The widget stays in the tree while the exit animation plays, then `onComplete` removes it. Without this, removing the widget immediately kills the animation mid-frame.

## Gotcha: initial state

Here's the catch — when `target` starts at `0.0`, the widget renders at the beginning of the effect chain (which is often `opacity: 0` if you have `fadeIn`). That's usually what you want. But if you start with `target: 1.0`, the widget renders fully visible from the first frame with no animation playing, which is also correct.

The problem is when you conditionally render the widget and then immediately set `target: 1.0`. Flutter builds the widget with `target: 1.0` on the first frame, so there's nothing to animate from — the widget just appears. If you want an entrance animation on conditional render, start `target` at `0.0` in `initState`, then set it to `1.0` in a post-frame callback:

```dart
double _target = 0.0;

@override
void initState() {
  super.initState();
  WidgetsBinding.instance.addPostFrameCallback((_) {
    setState(() => _target = 1.0);
  });
}

Widget build(BuildContext context) {
  return MyWidget()
    .animate(target: _target)
    .fadeIn(duration: 350.ms);
}
```

Ugly but necessary. Spent 20 minutes on this the first time.

## When to use target vs autoPlay

A quick rule of thumb:
- **autoPlay** (the default): animations that run once on first render — loading screens, page transitions, list items appearing
- **target**: animations driven by user interaction or state — toggles, tabs, drawers, accordion panels, step-by-step onboarding

Mixing them on the same widget causes confusing behavior. Pick one approach per widget.

---

Next in this series: using `flutter_animate` adapters — syncing animations to scroll position, value notifiers, and custom controllers for more advanced coordination.

**References:**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [flutter_animate README — Adapters and Target](https://pub.dev/packages/flutter_animate#adapters)
- [Previous post: staggered list animations with flutter_animate]({% post_url 2026-06-24-10-00-00-371817-flutter-animate-staggered-list %})
