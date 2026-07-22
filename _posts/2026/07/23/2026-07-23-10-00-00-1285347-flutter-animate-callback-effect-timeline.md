---
layout: post
title: "flutter_animate CallbackEffect - Trigger Flutter State Changes at a Precise Timeline Point"
description: "Learn how to use flutter_animate CallbackEffect to trigger one-shot state changes at an exact animation timeline point without duplicate callbacks."
date: 2026-07-23
tags: [flutter_animate, animation, state_management]
comments: true
share: true
---

![flutter_animate CallbackEffect timeline showing a midpoint callback](/assets/images/flutter-animate-callback-effect-timeline.png)

`CallbackEffect` is the cleanest way to run a side effect at a specific point in a `flutter_animate` sequence. It is useful when a card should update its label halfway through an entrance animation, a sound should play when a panel reaches its settled position, or analytics should record a visual milestone. The important detail is that the callback belongs to the animation timeline, not to a random `Future.delayed`.

## Why `Future.delayed` becomes fragile

I initially handled midpoint behavior with a delay matching the animation duration. That looked simple, but it broke as soon as the duration or curve changed. Rebuilding the widget could also schedule another delayed callback, and reversing or repeating the animation made the visual state and side effect disagree.

`CallbackEffect` keeps the event beside the effect that causes it. The timeline is easier to read and the callback receives a boolean telling you whether the animation is currently reversing.

| Approach | Timing source | Common failure |
|---|---|---|
| `Future.delayed` | Separate wall-clock timer | Drifts when duration changes |
| `onComplete` | End of the whole sequence | Cannot target a midpoint |
| `CallbackEffect` | Animation timeline | Must guard repeat/reverse cases |

The image above shows the useful mental model: start the sequence, place the callback marker, then finish the remaining effects.

## Add a callback between effects

The following widget changes its status once the fade reaches its midpoint. The `callback` extension is easier to scan than constructing `CallbackEffect` directly.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class UploadStatus extends StatefulWidget {
  const UploadStatus({super.key, required this.onHalfway});

  final VoidCallback onHalfway;

  @override
  State<UploadStatus> createState() => _UploadStatusState();
}

class _UploadStatusState extends State<UploadStatus> {
  bool _callbackWasRun = false;

  @override
  Widget build(BuildContext context) {
    return Text('Uploading...')
        .animate(onPlay: (_) => _callbackWasRun = false)
        .fadeIn(duration: 600.ms)
        .callback(
          duration: 300.ms,
          callback: (isReversed) {
            if (isReversed || _callbackWasRun) return;
            _callbackWasRun = true;
            widget.onHalfway();
          },
        )
        .scale(
          begin: const Offset(0.96, 0.96),
          end: const Offset(1, 1),
          duration: 300.ms,
        );
  }
}
```

The callback is positioned after the 600 ms fade and occupies the next 300 ms slot, so it fires at the intended timeline point before the scale effect completes. The exact visual interpolation still comes from the animation controller; the callback is only a notification.

## The guard matters more than the callback

An animation can play more than once. `onPlay` resets the local guard for a new forward run, while `isReversed` prevents a reverse pass from sending the same business event again. Without those two checks, a repeated animation can increment a counter, show a snackbar, or start a request multiple times.

Keep the callback small. If it needs asynchronous work, call a method that owns loading and error state rather than awaiting inside the timeline callback. Also avoid calling `setState` here unless the state change is intentional; the callback may run while a parent rebuilds the widget.

## Quick checklist

- Put visual timing in the effect chain, not in `Future.delayed`.
- Use `onComplete` for the end and `callback` for an internal milestone.
- Check the reverse flag when the animation can run backward.
- Add a one-shot guard when repeats or rebuilds are possible.
- Keep network, navigation, and other business work behind a small method.

`CallbackEffect` is a small API, but it closes a common gap between “the animation looks finished” and “the application should react now.” Once the event is expressed on the same timeline, changing a duration no longer requires hunting for unrelated timers.
