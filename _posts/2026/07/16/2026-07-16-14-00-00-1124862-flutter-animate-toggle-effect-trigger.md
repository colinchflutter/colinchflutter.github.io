---
layout: post
title: "flutter_animate ToggleEffect - Trigger Animated Widgets at the Right Moment"
description: "Learn how to use flutter_animate ToggleEffect to switch AnimatedContainer and other stateful animations at a precise point in a declarative timeline."
date: 2026-07-16
tags: [flutter_animate, animation, Flutter, state_management]
comments: true
share: true
---

![A Flutter settings card changing between two visual states](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

`ToggleEffect` is useful when an animation needs to hand control to another Flutter animation. It does not interpolate a color, size, or position itself. Instead, its builder receives `true` before a timeline point and `false` after it. That small boolean boundary is enough to activate `AnimatedContainer`, `AnimatedSwitcher`, or any other widget that already knows how to animate between values.

## Why use a toggle instead of `target`?

`target` is the natural choice when the same `flutter_animate` effects should move forward and backward as application state changes. `ToggleEffect` solves a slightly different problem: run a timeline once, then change the input of a separate animated widget.

| Requirement | Better fit |
| --- | --- |
| Fade or scale the same widget | `target` with regular effects |
| Swap content at a known timeline point | `toggle()` |
| Let `AnimatedContainer` interpolate its own properties | `toggle()` + `AnimatedContainer` |
| Rebuild on every animation frame | `custom()` |

The distinction matters for a loading-to-success card. The outer effect can reveal the card, while the inner `AnimatedContainer` handles the color and size transition.

## A loading-to-success card

The following example keeps the state inside the animation timeline. The `1.ms` duration makes the toggle boundary effectively immediate; the visible motion belongs to `AnimatedContainer`.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class UploadStatus extends StatelessWidget {
  const UploadStatus({super.key});

  @override
  Widget build(BuildContext context) {
    return Animate()
        .toggle(
          duration: 1.ms,
          delay: 900.ms,
          builder: (_, completed, child) {
            return AnimatedContainer(
              duration: 450.ms,
              curve: Curves.easeOutCubic,
              padding: const EdgeInsets.all(16),
              decoration: BoxDecoration(
                color: completed ? Colors.green.shade50 : Colors.blue.shade50,
                borderRadius: BorderRadius.circular(completed ? 20 : 12),
              ),
              child: Row(
                children: [
                  Icon(
                    completed ? Icons.check_circle : Icons.cloud_upload,
                    color: completed ? Colors.green : Colors.blue,
                  ),
                  const SizedBox(width: 12),
                  Text(completed ? 'Upload complete' : 'Uploading...'),
                ],
              ),
            );
          },
        )
        .fadeIn(duration: 250.ms)
        .slideY(begin: .08, end: 0, duration: 250.ms);
  }
}
```

The `delay` belongs to `ToggleEffect`, so the card enters immediately but remains in its loading state for 900 milliseconds. Once the boolean flips, Flutter compares the two `AnimatedContainer` configurations and animates the changed properties. The `fadeIn` and `slideY` effects still belong to the outer `Animate`, so they run independently during entry.

## A practical boundary: `toggle()` is not app state

The boolean describes the animation position, not the server result. If the upload can fail, retry, or be cancelled, keep that truth in a model or `State` object and use `target` or a new animation run for those branches. Treating `ToggleEffect` as the source of truth makes a replay difficult: the builder can show “complete” even though the request has failed.

For a reusable status component, keep the two responsibilities explicit:

- application state decides whether the operation succeeded;
- `ToggleEffect` decides when the visual handoff occurs;
- `AnimatedContainer` decides how its properties interpolate.

This also prevents a common mistake: giving `ToggleEffect` a long duration and expecting it to animate the color. It only changes the boolean when its effect reaches the end. Put the visual duration on the nested animated widget.

## Short takeaway

Use `ToggleEffect` when a declarative `flutter_animate` timeline must trigger a widget with its own animation system. A tiny toggle duration creates a clean handoff, while `AnimatedContainer` or `AnimatedSwitcher` owns the actual transition. Keep business state outside the effect so retries and reversals remain predictable.

