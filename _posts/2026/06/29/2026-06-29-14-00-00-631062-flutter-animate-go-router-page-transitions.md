---
layout: post
title: "flutter_animate Page Transitions with go_router — Custom Enter and Exit Animations"
description: "Wire flutter_animate into go_router's CustomTransitionPage to add polished enter/exit animations across your Flutter app — no duplicated AnimationController code in every route."
date: 2026-06-29
tags: [flutter_animate, go_router, animation, navigation, Flutter]
comments: true
share: true
---

![Winding road through mountains at golden hour](https://images.unsplash.com/photo-1449965408869-eaa3f722e40d?w=800&q=80)

Every Flutter app eventually needs page transitions that aren't the default slide-from-right. The problem is that go_router and flutter_animate feel like they live in different worlds — go_router wants `CustomTransitionPage` and `Animation<double>` from Navigator, while flutter_animate wants to attach effects to widgets declaratively. Connecting the two took me a while to figure out, so here's the exact approach.

## Why MaterialPageRoute doesn't cut it

`MaterialPageRoute` gives you a single fade or slide. You can't chain flutter_animate effects onto it without wrapping everything in `StatefulWidget` and manually hooking into `AnimationController`. That's 40+ lines of boilerplate per route.

go_router's `CustomTransitionPage` is different. It exposes `transitionsBuilder`, which gives you direct access to the `Animation<double>` that drives the route transition. You can pass that animation into flutter_animate's `Animate.fromValueListenable()` and chain any effect you want.

Short answer: `CustomTransitionPage` is the bridge.

## Setting up go_router with CustomTransitionPage

go_router version 13+ (current as of mid-2026). The key is defining a helper that wraps any widget in a `CustomTransitionPage`:

```dart
import 'package:flutter_animate/flutter_animate.dart';
import 'package:go_router/go_router.dart';

CustomTransitionPage<void> buildTransitionPage({
  required BuildContext context,
  required GoRouterState state,
  required Widget child,
}) {
  return CustomTransitionPage<void>(
    key: state.pageKey,
    child: child,
    transitionDuration: 400.ms,
    reverseTransitionDuration: 300.ms,
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      return _AnimatedRoute(
        animation: animation,
        secondaryAnimation: secondaryAnimation,
        child: child,
      );
    },
  );
}
```

`secondaryAnimation` is the outgoing route's animation — it fires when you push *on top* of the current screen, animating the current page out. Most tutorials ignore it, which is why the exit animation looks wrong on pop.

## The animated route widget

```dart
class _AnimatedRoute extends StatelessWidget {
  final Animation<double> animation;
  final Animation<double> secondaryAnimation;
  final Widget child;

  const _AnimatedRoute({
    required this.animation,
    required this.secondaryAnimation,
    required this.child,
  });

  @override
  Widget build(BuildContext context) {
    // Enter: slide up + fade in
    final entering = Animate.fromValueListenable(
      animation,
      child: child,
    ).slideY(begin: 0.04, end: 0, curve: Curves.easeOutCubic).fade();

    // Exit: scale down + fade out when a new route pushes on top
    return Animate.fromValueListenable(
      ReverseAnimation(secondaryAnimation),
      child: entering,
    ).scaleXY(begin: 1.0, end: 0.96, curve: Curves.easeIn).fade(
          begin: 1.0,
          end: 0.7,
        );
  }
}
```

`Animate.fromValueListenable()` takes an existing `Animation<double>` and uses it as the controller — so the animation is driven by Navigator, not by a separate timer. When the user swipes back, Navigator reverses the animation, and flutter_animate follows automatically.

The `ReverseAnimation` wrapper on `secondaryAnimation` is the trick. `secondaryAnimation` goes 0→1 when the screen is being covered (exit), but flutter_animate's `begin→end` assumes forward direction. Reversing it makes `begin` happen when the screen is covered and `end` when it's uncovered.

![App navigation flow diagram showing route transitions](https://images.unsplash.com/photo-1497366216548-37526070297c?w=800&q=80)

## Wiring it into GoRouter

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      pageBuilder: (context, state) => buildTransitionPage(
        context: context,
        state: state,
        child: const HomeScreen(),
      ),
    ),
    GoRoute(
      path: '/detail/:id',
      pageBuilder: (context, state) => buildTransitionPage(
        context: context,
        state: state,
        child: DetailScreen(id: state.pathParameters['id']!),
      ),
    ),
    GoRoute(
      path: '/settings',
      pageBuilder: (context, state) => buildTransitionPage(
        context: context,
        state: state,
        child: const SettingsScreen(),
      ),
    ),
  ],
);
```

Every route uses `pageBuilder` instead of `builder`. The `buildTransitionPage` helper handles the animation consistently. If you later want to change the transition style, you edit one function.

## Per-route override

Some routes need a different feel — a modal sheet that slides up from the bottom, for example:

```dart
CustomTransitionPage<void> buildBottomSheetTransition({
  required BuildContext context,
  required GoRouterState state,
  required Widget child,
}) {
  return CustomTransitionPage<void>(
    key: state.pageKey,
    child: child,
    transitionDuration: 350.ms,
    reverseTransitionDuration: 250.ms,
    opaque: false, // transparent background for sheet effect
    barrierColor: Colors.black54,
    barrierDismissible: true,
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      return Animate.fromValueListenable(animation, child: child)
          .slideY(begin: 1.0, end: 0, curve: Curves.easeOutQuart)
          .fade(begin: 0.5);
    },
  );
}
```

`opaque: false` + `barrierColor` is what makes the background page show through. Without `opaque: false`, the route fills the entire screen with white behind your widget.

## The iOS swipe-back problem

On iOS, the system swipe-back gesture drives `animation` directly. This works fine as long as your transition doesn't use `begin` values that make the screen temporarily invisible during a half-cancelled swipe.

The issue I ran into: setting `fade(begin: 0, end: 1)` on entry means that if the user starts a swipe-back and cancels it, the page snaps from half-visible to full opacity instead of smoothly returning. Fix: use `curve: Curves.easeOutCubic` on everything and keep the `begin` opacity above 0.4:

```dart
// Safe for iOS swipe-back
.fade(begin: 0.4, end: 1.0, curve: Curves.easeOutCubic)

// Problematic — snaps on cancelled swipe
.fade(begin: 0.0, end: 1.0)
```

Also, `transitionDuration` above 500ms makes the iOS swipe feel sluggish because the page trails behind the finger. 400ms is the safe upper limit.

## Handling shell routes

If you use `ShellRoute` for a bottom navigation bar, the shell itself doesn't get a transition — only the nested routes do. The shell widget stays mounted while routes swap inside it. flutter_animate effects on the shell's children animate independently from go_router's transition system, which is what you want.

```dart
ShellRoute(
  builder: (context, state, child) => MainShell(child: child),
  routes: [
    GoRoute(
      path: '/home',
      pageBuilder: (context, state) => buildTransitionPage(
        context: context,
        state: state,
        child: const HomeTab(),
      ),
    ),
    // ...
  ],
)
```

The `MainShell` with its `BottomNavigationBar` animates separately. Tab switches inside the shell show the page transition; the shell itself stays still.

---

Next: flutter_animate with `AnimationController` — when you need to manually control playback, reverse mid-animation, or synchronize multiple widgets to a single shared controller.

**References:**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [go_router CustomTransitionPage API](https://pub.dev/documentation/go_router/latest/go_router/CustomTransitionPage-class.html)
- [Flutter page transition animation docs](https://docs.flutter.dev/cookbook/animation/page-route-animation)
