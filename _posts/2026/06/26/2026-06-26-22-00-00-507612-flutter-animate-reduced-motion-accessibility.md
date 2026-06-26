---
layout: post
title: "flutter_animate and Reduced Motion: Building Accessible Flutter Apps"
description: "Learn how to respect iOS and Android reduced motion settings in flutter_animate — conditionally simplifying or disabling animations without rewriting every widget."
date: 2026-06-26
tags: [flutter_animate, animation, accessibility, Android, iOS]
comments: true
share: true
---

# flutter_animate and Reduced Motion: Building Accessible Flutter Apps

![Smartphone accessibility settings screen showing reduce motion toggle](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

`flutter_animate` makes it easy to add polished animations, but there's a catch: some users have "Reduce Motion" turned on in their system settings. On iOS, that disables parallax and crossfade transitions. On Android, it collapses the animation scale to 0. If your flutter_animate animations ignore those settings, you're breaking accessibility for anyone who gets motion sick, has vestibular disorders, or simply prefers less visual noise.

The fix isn't complicated, but it took me longer than I'd like to admit to wire it up properly.

## What "Reduce Motion" actually means per platform

On **iOS 17+**, Settings → Accessibility → Motion → Reduce Motion. When enabled, `MediaQuery.of(context).disableAnimations` returns `true` in Flutter.

On **Android**, it's the "Remove animations" option under Developer Options (Animation Scale set to 0). Same result — `disableAnimations` becomes `true`.

The key insight is that Flutter already surfaces this via `MediaQuery`. You don't need any native channel or plugin. What you do need is to actually check it before running your animations.

## The naive approach that technically works

The simplest thing: wrap every animated widget in a check.

```dart
Widget build(BuildContext context) {
  final noMotion = MediaQuery.of(context).disableAnimations;

  return noMotion
      ? const SizedBox(width: 200, height: 60, child: MyCard())
      : MyCard()
          .animate()
          .fadeIn(duration: 400.ms)
          .slideY(begin: 0.3, end: 0);
}
```

Works. But you'll repeat this in every widget that has an animation. Not great.

## A cleaner wrapper: AnimateIf

Here's the pattern I landed on — a small wrapper that reads `disableAnimations` and decides whether to apply effects at all:

```dart
class AnimateIf extends StatelessWidget {
  const AnimateIf({
    super.key,
    required this.child,
    required this.effects,
    this.onPlay,
  });

  final Widget child;
  final List<Effect> effects;
  final void Function(AnimationController)? onPlay;

  @override
  Widget build(BuildContext context) {
    if (MediaQuery.of(context).disableAnimations) {
      return child;
    }
    return child.animate(onPlay: onPlay, effects: effects);
  }
}
```

Usage is almost identical to the normal flutter_animate API:

```dart
AnimateIf(
  effects: [
    FadeEffect(duration: 400.ms),
    SlideEffect(begin: const Offset(0, 0.3), end: Offset.zero),
  ],
  child: MyCard(),
)
```

If the user has reduced motion enabled, you get `MyCard()` with no animation. Otherwise, the full chain runs.

## Global toggle via a provider

If you want to go further — maybe you have your own in-app "disable animations" setting on top of the system one — a Riverpod/Provider approach works well:

```dart
// With Riverpod (flutter_riverpod ^2.x)
final motionReducedProvider = Provider<bool>((ref) {
  // Resolved in a widget: ref.watch(motionReducedProvider)
  // Override in tests by setting ProviderScope overrides
  return false; // actual value read from MediaQuery at widget level
});
```

Honestly, for most apps reading from `MediaQuery` directly is fine. Only reach for a provider if you're adding a user-facing "reduce motion" toggle inside the app itself.

## The parts that actually confused me

**`disableAnimations` vs `highContrast` vs `boldText`**: They're all separate flags. `disableAnimations` is the one for motion. I initially grabbed the wrong one.

**Simulator vs real device**: In the iOS Simulator you can toggle Reduce Motion from Hardware → Toggle Increased Contrast... wait, that's the wrong menu. It's actually Features → Toggle Reduced Motion (or `xcrun simctl status_bar`). On a real iPhone, go to Settings → Accessibility → Motion. The simulator flag doesn't always match physical device behavior — test on a real device before shipping.

**Android Developer Options gotcha**: If "Remove animations" is enabled *and* then you disable Developer Mode entirely, the animation scale resets to 1x. A user who can't find Developer Options won't be able to reproduce the issue when you debug it with them.

```dart
// Quick debug helper — remove before prod
void _debugMotion(BuildContext context) {
  final mq = MediaQuery.of(context);
  debugPrint('disableAnimations: ${mq.disableAnimations}');
  debugPrint('accessibleNavigation: ${mq.accessibleNavigation}');
}
```

## Testing it in widget tests

You can override `MediaQueryData` to simulate reduced motion without needing a real device:

```dart
testWidgets('skips animation when reduce motion enabled', (tester) async {
  await tester.pumpWidget(
    MediaQuery(
      data: const MediaQueryData(disableAnimations: true),
      child: MaterialApp(
        home: Scaffold(body: AnimateIf(
          effects: [FadeEffect(duration: 400.ms)],
          child: const Text('Hello'),
        )),
      ),
    ),
  );

  // Text should be present immediately, no animation frames needed
  expect(find.text('Hello'), findsOneWidget);

  // With animation: would need pump(400.ms) for full visibility
  // Without: already at full opacity from frame 0
  final opacity = tester.widget<Opacity>(find.byType(Opacity));
  // No Opacity wrapper expected — the child is returned as-is
  expect(find.byType(Animate), findsNothing);
});
```

The `AnimateIf` approach makes this trivial to test: if `disableAnimations` is true, no `Animate` widget exists in the tree at all.

---

Next up: writing widget tests directly against flutter_animate effects — how to verify specific animation values at given time offsets using `tester.pump(duration)`.

**References**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [Flutter accessibility docs — disableAnimations](https://api.flutter.dev/flutter/widgets/MediaQueryData/disableAnimations.html)
- [Flutter PR #180041 — reduced motion on web](https://github.com/flutter/flutter/pull/180041)
