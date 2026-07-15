---
layout: post
title: "flutter_animate CustomEffect - Build Effects for UI States the Package Does Not Know"
description: "Learn how to use flutter_animate CustomEffect to animate progress, status colors, and custom layout values without writing a separate AnimationController."
date: 2026-07-15
tags: [flutter_animate, animation, Flutter, performance]
comments: true
share: true
---

![Flutter animation timeline with a custom progress indicator](https://images.unsplash.com/photo-1558655146-d09347e92766?w=1200&q=80)

`flutter_animate` already covers fade, scale, blur, shimmer, and color changes. The moment the UI needs a value the package does not have a named effect for, `CustomEffect` is the escape hatch. It gives a builder a normalized value, so a progress bar, status badge, or custom layout can animate inside the same declarative chain without creating another `AnimationController`.

## The value is the important part

The builder receives `BuildContext`, `value`, and `child`. By default, `value` moves from `0.0` to `1.0` during the effect. `begin` and `end` let the value represent something more useful, such as 0 to 100 percent.

| Use case | `begin` → `end` | Builder output |
| --- | ---: | --- |
| Progress bar | `0` → `100` | `LinearProgressIndicator` |
| Score counter | `0` → `850` | `Text` |
| Status color | `0` → `1` | `Color.lerp` |
| Card offset | `-24` → `0` | `Transform.translate` |

The builder runs as the animation ticks, so it must stay cheap. Keep network calls, logging, and state mutations out of it.

## A progress bar with a number that agrees

This example animates both the bar and its label from the same `value`. Using one value avoids the small mismatch that appears when two independent animations have different curves or durations.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class UploadProgress extends StatelessWidget {
  const UploadProgress({super.key});

  @override
  Widget build(BuildContext context) {
    return Animate()
        .custom(
          begin: 0,
          end: 100,
          duration: 1200.ms,
          curve: Curves.easeOutCubic,
          builder: (context, value, child) {
            final percent = value.clamp(0, 100).round();
            return Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                Text('$percent% uploaded'),
                const SizedBox(height: 8),
                LinearProgressIndicator(value: percent / 100),
              ],
            );
          },
        )
        .fadeIn();
  }
}
```

`Animate()` without a child is intentional. `CustomEffect` becomes the widget builder, and the following `fadeIn()` still applies to the widget returned by the custom builder. This is handy for generated content such as a number that did not exist before the animation started.

## Keep the animated child inside the builder

When decorating an existing widget, return `child` from the builder. Omitting it silently removes the target from the result.

```dart
final status = Text('Syncing...').animate().custom(
  duration: 700.ms,
  begin: 0,
  end: 1,
  builder: (_, value, child) {
    final color = Color.lerp(Colors.orange, Colors.green, value)!;
    return DecoratedBox(
      decoration: BoxDecoration(
        color: color.withValues(alpha: .12),
        borderRadius: BorderRadius.circular(8),
      ),
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
        child: child,
      ),
    );
  },
);
```

The `child` may already include effects placed earlier in the chain. That makes `CustomEffect` composable, but it also means the builder should not rebuild a large page around it. Extract the animated region into a small widget when the effect sits in a dense list.

## Three traps that caused real bugs

First, do not assume `value` always stays between zero and one. A curve can overshoot. Clamp values before passing them to APIs such as `LinearProgressIndicator` or before converting them to an integer counter.

Second, do not use `setState` to mirror the builder value. The builder is already rebuilt by the animation. Calling `setState` from it creates a rebuild loop and can throw when the widget is disposed during a route change.

Third, test the timing of chained effects. A custom effect with no explicit duration inherits the chain timing, while an explicit `delay` starts it later. Write the intended timeline down before stacking `custom()`, `fadeIn()`, and `then()`.

`CustomEffect` is the right choice when the missing feature is a visual mapping from one number to widgets. If the animation must react to a stream, scroll position, or external model, use an adapter and let that source control the animation value instead.

The API details used here are documented in the [flutter_animate CustomEffect reference](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/CustomEffect-class.html) and the [package guide](https://pub.dev/packages/flutter_animate).
