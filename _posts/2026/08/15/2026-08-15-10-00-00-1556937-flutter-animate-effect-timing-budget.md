---
layout: post
title: "flutter_animate Effect Timing - Prevent Clipped Flutter Animation Sequences"
description: "Learn how to budget delay, duration, and interval in flutter_animate so chained Flutter effects finish cleanly without clipped or frozen transitions."
date: 2026-08-15
tags: [flutter_animate, animation, performance, Flutter]
comments: true
share: true
---

![Flutter animation timeline with delay and duration budgets](/assets/images/flutter-animate-curves-motion.png)

*The useful part of this timeline is the finish point: every delayed effect must fit inside the controller's total animation window.*

`flutter_animate` makes effect chains look deceptively simple. I can write `.fadeIn().slideY()` and get a working transition immediately. The trouble starts when a card has a 200 ms delay, a 400 ms slide, and a parent controller that lasts only 450 ms. The last effect is not broken; it is being cut off by the timeline.

The reliable fix is to treat every effect as a time range and budget the whole sequence before tuning curves.

## The timing rule that prevents clipped effects

For one effect, the end time is:

`delay + duration`

For a chain, the animation must stay within the longest end time. A small timing table makes the problem obvious:

| Effect | Delay | Duration | End time |
|---|---:|---:|---:|
| Fade in | 0 ms | 220 ms | 220 ms |
| Slide up | 80 ms | 360 ms | 440 ms |
| Scale emphasis | 260 ms | 280 ms | 540 ms |

The sequence needs at least 540 ms. A controller duration of 400 ms will show the fade and part of the slide, then stop before the scale effect reaches its end value.

## Build a card with an explicit timing budget

The code below keeps the offsets visible instead of scattering unexplained numbers across the widget tree:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class ResultCard extends StatelessWidget {
  const ResultCard({super.key});

  static const fadeDuration = Duration(milliseconds: 220);
  static const slideDelay = Duration(milliseconds: 80);
  static const slideDuration = Duration(milliseconds: 360);
  static const scaleDelay = Duration(milliseconds: 260);
  static const scaleDuration = Duration(milliseconds: 280);

  // max(delay + duration) = 540 ms.
  static const totalDuration = Duration(milliseconds: 540);

  @override
  Widget build(BuildContext context) {
    return Card(
      child: const ListTile(
        leading: Icon(Icons.check_circle_outline),
        title: Text('Backup completed'),
        subtitle: Text('Your latest snapshot is ready.'),
      ),
    )
        .animate()
        .fadeIn(duration: fadeDuration)
        .slideY(
          begin: .14,
          end: 0,
          delay: slideDelay,
          duration: slideDuration,
          curve: Curves.easeOutCubic,
        )
        .scaleXY(
          begin: .98,
          end: 1,
          delay: scaleDelay,
          duration: scaleDuration,
          curve: Curves.easeOut,
        );
  }
}
```

There is no need to set a controller for a normal entrance animation. The important decision is that the largest `delay + duration` is deliberate. If the design changes and the scale delay moves to 400 ms, the total budget must move to at least 680 ms as well.

## Why a chain can feel sequential when it is not

Effects in a `flutter_animate` chain are normally placed on one shared timeline. A delay does not mean “wait until the previous effect finishes.” It means “start this effect at this absolute offset.” That difference matters:

```dart
// Overlapping timeline: slide starts at 80 ms while fade is still running.
widget
    .animate()
    .fadeIn(duration: 220.ms)
    .slideY(delay: 80.ms, duration: 360.ms);
```

This overlap usually feels better for a card. If the second effect must start after the first one, calculate its delay from the previous end time instead:

```dart
widget
    .animate()
    .fadeIn(duration: 220.ms)
    .slideY(delay: 220.ms, duration: 360.ms);
```

The second version ends at 580 ms, not 360 ms. It is a true sequence, but it is also slower. I initially increased every delay to make the motion “clearer,” then the UI started feeling sluggish on repeated refreshes. Overlap is often the better choice for small state changes.

## When an external controller changes the calculation

With an external `AnimationController`, its duration becomes the ceiling for every attached `Animate` widget. Keep the controller at least as long as the largest effect end time:

```dart
late final AnimationController _controller = AnimationController(
  vsync: this,
  duration: const Duration(milliseconds: 540),
);

Animate(
  autoPlay: false,
  controller: _controller,
  effects: const [
    FadeEffect(duration: Duration(milliseconds: 220)),
    SlideEffect(
      delay: Duration(milliseconds: 80),
      duration: Duration(milliseconds: 360),
    ),
  ],
  child: const ResultCardContent(),
);
```

Do not solve a clipped animation by making the controller excessively long. A 1,500 ms controller for a 540 ms card makes the same effects move more slowly or leaves a long settled tail, depending on how the timeline is configured. Set the ceiling from the actual end time, then add a small design margin only if the last frame needs breathing room.

## A practical debugging checklist

When an effect appears frozen, check the timeline before changing the curve:

| Check | What to look for |
|---|---|
| Start offset | Is the delay later than expected? |
| End time | Does `delay + duration` exceed the controller duration? |
| Rebuild behavior | Did a new key restart the animation unexpectedly? |
| List items | Are many off-screen effects consuming the same frame budget? |

The most useful temporary trick is to replace the final effect with a conspicuous color or scale change. If that change never appears, the issue is probably timing or lifecycle, not the visual effect itself.

`flutter_animate` handles the interpolation; the developer still owns the schedule. Write down the end times for the two or three effects that matter, choose overlap or true sequencing intentionally, and make the controller duration cover the latest endpoint. That small bit of arithmetic removes a surprising number of “random” Flutter animation bugs.
