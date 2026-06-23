---
layout: post
title: "flutter_animate introduction — animations without the boilerplate"
description: "flutter_animate turns Flutter's verbose AnimationController setup into chainable method calls. This post covers the basics: fadeIn, slideX, scale, chaining effects, and delay — with working code you can drop straight in."
date: 2026-06-23
tags: [flutter_animate, Flutter, animation, Dart]
comments: true
share: true
---

# flutter_animate introduction — animations without the boilerplate

![Flutter animation](https://images.unsplash.com/photo-1550063873-ab792950096b?w=800&q=80)

Flutter animations have always worked fine. They just take a lot of code to set up. An `AnimationController`, a `Ticker`, an `AnimatedBuilder` or `AnimatedWidget`, the actual `Tween` — that's four moving parts before a single widget fades in. `flutter_animate` (version 4.x) collapses all of that into extension methods chained directly on the widget.

## Setup

```yaml
# pubspec.yaml
dependencies:
  flutter_animate: ^4.5.0
```

No initialization required. Import and use.

```dart
import 'package:flutter_animate/flutter_animate.dart';
```

## Basic usage

The package adds `.animate()` to every widget. Chain effects after it.

```dart
// Before: static text
Text('Hello')

// After: text that fades in over 400ms
Text('Hello').animate().fadeIn()
```

`fadeIn()` defaults to 300ms with a linear curve. That's usually too abrupt — bump the duration:

```dart
Text('Hello')
  .animate()
  .fadeIn(duration: 600.ms, curve: Curves.easeOut)
```

The `.ms` extension on `int` is a nice touch. `600.ms` reads better than `Duration(milliseconds: 600)` inline.

## Chaining effects

Effects chain left to right. Each one runs after the previous by default.

```dart
Container(color: Colors.blue, width: 100, height: 100)
  .animate()
  .fadeIn(duration: 400.ms)
  .slideX(begin: -0.3, end: 0, duration: 400.ms)
  .scale(begin: Offset(0.8, 0.8), end: Offset(1, 1))
```

That gives you fade + slide from left + scale up, all in sequence. The total play time is the sum of each effect's duration.

## Running effects in parallel

Add `delay: 0.ms` to make an effect start at the same time as the previous one:

```dart
Text('Hello')
  .animate()
  .fadeIn(duration: 500.ms)
  .slideY(begin: 0.2, end: 0, duration: 500.ms, delay: 0.ms)
```

Both effects start immediately and run together.

## Delay

To stagger a list of items, use `delay` on the `.animate()` call itself:

```dart
Column(
  children: [
    for (int i = 0; i < items.length; i++)
      ListTile(title: Text(items[i]))
        .animate(delay: (100 * i).ms)
        .fadeIn()
        .slideX(begin: 0.1, end: 0),
  ],
)
```

Each item waits an extra 100ms. Works cleanly without a `ListView.builder`.

## The part that tripped me up

By default, the animation plays once when the widget first builds. That's fine for entry animations. But if you rebuild the widget (state change, navigation pop, etc.), the animation does **not** replay. The widget tree thinks it already ran.

If you need the animation to replay on rebuild, pass `onPlay`:

```dart
Text('Hello')
  .animate(onPlay: (controller) => controller.repeat())
  .shimmer()
```

Or trigger it manually with a key:

```dart
// Changing the key forces a full rebuild including animation replay
Text('Hello')
  .animate(key: ValueKey(triggerCount))
  .fadeIn()
```

This caught me off guard with a loading indicator that stopped animating after the first data refresh.

## What's next

The next post covers `flutter_animate` with `CustomPainter` — drawing paths that animate on entry. The package has a `custom` effect builder that plugs into any value you want to drive.

---

Next: animating custom-drawn widgets with flutter_animate's `CustomEffect`.
