---
layout: post
title: "flutter_animate ToggleEffect — Switch Flutter Widget States at the Right Time"
description: "Use flutter_animate ToggleEffect to swap a Flutter widget's visual state during a timeline, with looping examples and timing pitfalls explained."
date: 2026-08-11
tags: [flutter_animate, animation, state_management, Flutter, Dart]
comments: true
share: true
---

![flutter_animate ToggleEffect switching a card between two visual states](https://colinchflutter.github.io/assets/images/flutter-animate-toggle-effect-state-switching.png)

*The useful idea is the two-state handoff: let the animation timeline decide when the widget changes, instead of adding a separate timer or boolean flag.*

`flutter_animate`'s `ToggleEffect` is for a different job than `FadeEffect` or `ScaleEffect`. It does not give your builder a smoothly changing `double`; it gives it a `bool` that flips at the effect's transition point. That makes it a good fit for an active/inactive card, a loading/ready label, or an icon that should change halfway through a loop.

## The basic ToggleEffect

The builder receives `value` and the child that existed before `.animate()`. The child is still useful when only one property should change.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class SyncCard extends StatelessWidget {
  const SyncCard({super.key});

  @override
  Widget build(BuildContext context) {
    return Container(
      width: 240,
      padding: const EdgeInsets.all(20),
      decoration: BoxDecoration(
        color: Colors.indigo.shade50,
        borderRadius: BorderRadius.circular(18),
      ),
      child: const Text('Syncing data...'),
    ).animate(
      onPlay: (controller) => controller.repeat(reverse: true),
    ).toggle(
      duration: 900.ms,
      builder: (_, isReady, _) {
        return AnimatedContainer(
          duration: 250.ms,
          padding: const EdgeInsets.all(20),
          decoration: BoxDecoration(
            color: isReady ? Colors.green.shade50 : Colors.indigo.shade50,
            borderRadius: BorderRadius.circular(18),
            border: Border.all(
              color: isReady ? Colors.green : Colors.indigo,
            ),
          ),
          child: Text(isReady ? 'Synced' : 'Syncing data...'),
        );
      },
    );
  }
}
```

The `ToggleEffect` itself changes the boolean at the end of its timeline. `AnimatedContainer` then softens the visual change. This separation matters: `ToggleEffect` chooses *when* the state changes, while `AnimatedContainer` chooses *how* the new layout settles.

## Why reverse looping matters

`repeat(reverse: true)` is easy to overlook. A plain `repeat()` restarts the controller from zero, so the toggle can repeatedly enter the same half of the timeline. Reversing the controller gives the effect a predictable back-and-forth cycle.

| Goal | Better choice | Reason |
| --- | --- | --- |
| Switch once during an entrance animation | `toggle()` | No extra state variable |
| Repeat two visual states | `toggle()` + `repeat(reverse: true)` | The controller travels both directions |
| Animate a numeric value continuously | `CustomEffect` | The builder receives a `double` |
| React to business state | `animate(target:)` | The state change comes from your model |

There is another trap in the example. The boolean is not a replacement for application state. If the card must remain “Synced” after a network request finishes, store that fact in your model and use `target:` or a normal state rebuild. `ToggleEffect` is timeline state, so it can change again when the animation repeats.

## Keeping the child and layout stable

The builder can replace the child entirely, but changing text length or padding at the same instant may produce a layout jump. Give both states the same outer constraints, or put the changing content inside a fixed-width `SizedBox`. For icons, use `AnimatedSwitcher` inside the toggle builder when the old and new icons need their own crossfade.

Also avoid using `ToggleEffect` to fake a continuous progress indicator. Its callback is intentionally boolean. If you need a progress value for a custom painter, shader, or gradient, use `CustomEffect` or `ListenEffect` instead.

`ToggleEffect` is small, but it removes a surprisingly common piece of animation bookkeeping. Use it when the important event is a discrete handoff on an animation timeline, keep durable state outside the effect, and let a second widget handle the visual interpolation.

Official references: [ToggleEffect API](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/ToggleEffect-class.html) and [flutter_animate package](https://pub.dev/packages/flutter_animate).
