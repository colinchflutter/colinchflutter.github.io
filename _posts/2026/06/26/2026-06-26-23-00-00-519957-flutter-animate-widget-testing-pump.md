---
layout: post
title: "flutter_animate Widget Testing: Verifying Effects with tester.pump"
description: "Learn how to write Flutter widget tests for flutter_animate effects — assert opacity, position, and scale values at specific time offsets using tester.pump(duration)."
date: 2026-06-26
tags: [flutter_animate, testing, animation, Flutter]
comments: true
share: true
---

![Developer writing tests on a laptop with code editor open](https://images.unsplash.com/photo-1607799279861-4dd421887fb3?w=800&q=80)

Most Flutter teams write zero tests for animations. Honestly, that's somewhat understandable — animations are time-based, visual, and Flutter's test framework requires knowing how time interacts with the widget tree. But with `flutter_animate`, the test surface is actually pretty clean once you know the two or three things that trip you up.

Short answer: `tester.pump(duration)` advances the test clock by that duration. Call it after triggering an animation, then inspect the widget's rendered properties to verify what the effect is doing at that point in time.

## The problem with `pumpAndSettle`

The first thing you'll try is `pumpAndSettle()`. It runs frames until the widget tree is idle. For most UI tests, that's exactly what you want. For animations, it's a footgun.

If you have any animation with `repeat: true` or an `Animate` widget with an ongoing `onPlay` callback that restarts itself, `pumpAndSettle()` will time out after 100 seconds because the tree never goes idle.

```dart
// This hangs if the animation loops
await tester.pumpAndSettle(); // ❌ timeout with looping animations
```

Use `tester.pump(duration)` instead. It gives you control over exactly when in the animation timeline you're sampling.

## Basic setup: testing a FadeIn effect

Here's a widget that fades in over 400ms when it appears:

```dart
class FadeCard extends StatelessWidget {
  const FadeCard({super.key});

  @override
  Widget build(BuildContext context) {
    return const Text('Hello')
        .animate()
        .fadeIn(duration: 400.ms);
  }
}
```

The test:

```dart
testWidgets('FadeCard fades in over 400ms', (tester) async {
  await tester.pumpWidget(
    const MaterialApp(home: Scaffold(body: FadeCard())),
  );

  // Frame 0: animation just started, opacity should be ~0
  final opacityStart = _getOpacity(tester);
  expect(opacityStart, lessThan(0.1));

  // Halfway through
  await tester.pump(200.ms);
  final opacityMid = _getOpacity(tester);
  expect(opacityMid, greaterThan(0.4));
  expect(opacityMid, lessThan(0.7));

  // End of animation
  await tester.pump(200.ms);
  final opacityEnd = _getOpacity(tester);
  expect(opacityEnd, closeTo(1.0, 0.01));
});

double _getOpacity(WidgetTester tester) {
  final fadeTransition = tester.widget<FadeTransition>(
    find.byType(FadeTransition).first,
  );
  return fadeTransition.opacity.value;
}
```

The key: flutter_animate internally uses Flutter's standard animation primitives (`FadeTransition`, `SlideTransition`, etc.), so you can walk the widget tree and read the `.value` on the underlying animations.

![Flutter widget test output showing animation frame values](https://images.unsplash.com/photo-1461749280684-dccba630e2f6?w=800&q=80)

## Pumping the first frame

There's a subtle timing issue. When you call `pumpWidget`, the widget is built but the animation hasn't ticked yet. The first `pump()` call with no arguments (`tester.pump()`) triggers a single frame:

```dart
await tester.pumpWidget(...);
// At this point: Animate exists but no frames have run
await tester.pump(); // Run frame 0 — animation starts here
```

Some effects start from opacity 0 at frame 0. If you skip this initial `pump()`, your assertions about the starting state might hit the wrong frame. I wasted an hour on this.

## Testing slide effects

`SlideEffect` maps to `SlideTransition`. Here's how you inspect the position:

```dart
testWidgets('slides up from below', (tester) async {
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: const Text('Item')
            .animate()
            .slideY(begin: 0.5, end: 0, duration: 300.ms),
      ),
    ),
  );

  await tester.pump(); // start animation

  // Initial position: 50% below
  final slideStart = _getSlideY(tester);
  expect(slideStart, closeTo(0.5, 0.05));

  await tester.pump(150.ms); // halfway
  final slideMid = _getSlideY(tester);
  expect(slideMid, closeTo(0.25, 0.1)); // ~halfway between 0.5 and 0

  await tester.pump(150.ms); // end
  final slideEnd = _getSlideY(tester);
  expect(slideEnd, closeTo(0.0, 0.01));
});

double _getSlideY(WidgetTester tester) {
  final slideTransition = tester.widget<SlideTransition>(
    find.byType(SlideTransition).first,
  );
  return slideTransition.position.value.dy;
}
```

## Testing staggered animations

Staggered lists add a `delay` to each item's `Animate`. The test for this is slightly different — you need to pump past the delay before the animation starts.

```dart
// Widget with 200ms delay
Text('Delayed').animate(delay: 200.ms).fadeIn(duration: 300.ms)
```

```dart
testWidgets('delayed animation does not start before delay', (tester) async {
  await tester.pumpWidget(...);
  await tester.pump();

  // At 100ms, the animation hasn't started yet
  await tester.pump(100.ms);
  final opacityBeforeDelay = _getOpacity(tester);
  expect(opacityBeforeDelay, lessThan(0.05)); // still transparent

  // At 250ms (past the 200ms delay), animation has started
  await tester.pump(150.ms);
  final opacityAfterDelay = _getOpacity(tester);
  expect(opacityAfterDelay, greaterThan(0.1));
});
```

## Testing chained effects with `.then()`

When you chain effects with `.then()`, each effect starts after the previous one finishes. The test needs to pump through the full duration of the first effect:

```dart
// Total: 400ms fade + 300ms slide = 700ms
Text('Chained')
    .animate()
    .fadeIn(duration: 400.ms)
    .then()
    .slideX(begin: -0.2, end: 0, duration: 300.ms);
```

```dart
testWidgets('slide starts after fade completes', (tester) async {
  await tester.pumpWidget(...);
  await tester.pump();

  // At 200ms: fade is 50% done, slide hasn't started
  await tester.pump(200.ms);
  final opacityMid = _getOpacity(tester);
  expect(opacityMid, closeTo(0.5, 0.1));

  // At 400ms: fade done, slide just started
  await tester.pump(200.ms);
  final opacityFull = _getOpacity(tester);
  expect(opacityFull, closeTo(1.0, 0.01));

  final slideStart = _getSlideX(tester);
  expect(slideStart, lessThan(-0.1)); // slide beginning at -0.2
});
```

## Where this breaks down

**Scale effects** don't use a standard Flutter transition widget — `flutter_animate` sometimes applies them through a `Transform` widget. You can still find and inspect them, but you need `find.byType(Transform)` instead of a named transition class.

**Curve accuracy**: The default curve for most effects is `Curves.linear` unless you specify otherwise. If you're testing at the midpoint and expecting exactly `0.5`, make sure the curve is actually linear. `Curves.easeInOut` will give you ~0.5 at the midpoint but the curve shape means intermediate values won't be exactly linear. Set tolerances accordingly.

**Multiple Animate widgets**: If you have several animated widgets in the tree, `find.byType(FadeTransition).first` grabs the first one. Be explicit with `find.descendant(of: find.byKey(...), matching: find.byType(FadeTransition))` when you have more than one animated widget on screen.

## Golden tests for visual verification

If you want to capture the exact visual output at a specific frame rather than inspecting widget properties, golden tests work well here:

```dart
testWidgets('animation at 50% golden', (tester) async {
  await tester.pumpWidget(...);
  await tester.pump(200.ms); // halfway through a 400ms animation

  await expectLater(
    find.byType(MaterialApp),
    matchesGoldenFile('goldens/fade_card_50pct.png'),
  );
});
```

Run `flutter test --update-goldens` once to capture the reference images, then subsequent runs will fail if the animation output changes. Useful for catching unintended visual regressions when upgrading `flutter_animate` versions.

---

Next up: combining `flutter_animate` with `go_router` page transitions — custom enter/exit animations on route changes without fighting Flutter's built-in transition system.

**References**
- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [Flutter widget testing docs](https://docs.flutter.dev/testing/overview#widget-tests)
- [WidgetTester.pump API](https://api.flutter.dev/flutter/flutter_test/WidgetTester/pump.html)
