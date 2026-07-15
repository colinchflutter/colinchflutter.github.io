---
layout: post
title: "flutter_animate Sequenced Effects - Control Multi-Stage Motion with then()"
description: "Learn how to sequence flutter_animate effects with then(), delays, and repeat behavior without turning a simple Flutter UI into nested animation controllers."
date: 2026-07-16
tags: [flutter_animate, animation, Flutter, performance]
comments: true
share: true
---

![A Flutter interface showing a staged loading card animation](https://images.unsplash.com/photo-1558655146-d09347e92766?w=1200&q=80)

`flutter_animate` becomes much easier to reason about when each effect has a clear place on a timeline. Instead of creating one `AnimationController` for the entrance, another for the highlight, and a third for the idle pulse, chain the stages with `then()`. The result is a single declarative animation whose order is visible in the widget code.

## A sequence is different from a stack

Effects added directly to an `Animate` chain share the same overall timeline. That is useful when opacity and translation should happen together:

```dart
Card(
  child: const ListTile(
    leading: Icon(Icons.cloud_download),
    title: Text('Preparing your files'),
  ),
).animate()
    .fadeIn(duration: 280.ms)
    .slideY(begin: .08, end: 0, duration: 280.ms);
```

The card fades and moves at the same time. A staged interaction needs a different shape: enter first, draw attention to the status, then settle. `then()` starts a new section after the previous effect has finished.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class DownloadStatusCard extends StatelessWidget {
  const DownloadStatusCard({super.key});

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.all(24),
      child: const ListTile(
        leading: Icon(Icons.cloud_download),
        title: Text('Preparing your files'),
        subtitle: Text('This usually takes a few seconds'),
      ),
    )
        .animate()
        .fadeIn(duration: 280.ms)
        .slideY(begin: .08, end: 0, duration: 280.ms)
        .then(delay: 120.ms)
        .shimmer(duration: 650.ms, color: Colors.white54)
        .then(delay: 500.ms)
        .scale(
          begin: 1.0,
          end: 1.02,
          duration: 180.ms,
        )
        .then()
        .scale(
          begin: 1.02,
          end: 1.0,
          duration: 180.ms,
        );
  }
}
```

![Timeline of the staged flutter_animate effects](https://images.unsplash.com/photo-1550745165-9bc0b252726f?w=1200&q=80)

The important detail is that the second `scale` does not run beside the first one. It starts after the first scale completes because it follows another `then()`. The image above should be read as four blocks: entrance, pause, highlight, and settle.

## Read the timing before changing the curve

The total time is the sum of effect durations and explicit delays. Keeping that arithmetic visible prevents a small card animation from feeling sluggish.

| Stage | Duration | Delay before stage | Purpose |
| --- | ---: | ---: | --- |
| Fade and slide | 280 ms | 0 ms | Establish the card |
| Shimmer | 650 ms | 120 ms | Signal activity |
| Scale up | 180 ms | 500 ms | Add a short emphasis |
| Scale down | 180 ms | 0 ms | Return to rest |

In this example the first meaningful motion finishes after 280 ms. The shimmer begins 120 ms later, and the scale emphasis is deliberately delayed by 500 ms. If the card appears after a network response, that is usually enough feedback without making the user wait for the animation before seeing content.

For a list, do not put a long `then(delay: ...)` on every row unless the delay is intentional. A per-item entrance can be staggered with each child receiving a calculated delay, while each child keeps a short local sequence:

```dart
ListView.builder(
  itemCount: files.length,
  itemBuilder: (context, index) {
    return FileTile(file: files[index])
        .animate(delay: (index * 45).ms)
        .fadeIn(duration: 220.ms)
        .slideX(begin: .06, end: 0, duration: 220.ms);
  },
);
```

The list delay controls when a row enters. `then()` controls what that row does after entering. Mixing those responsibilities makes the timeline much easier to tune.

## Repeat only the part that should loop

A common mistake is putting `repeat()` at the end of a long entrance chain and expecting only the shimmer to repeat. The entire chain can replay, so the card repeatedly disappears and re-enters. For an idle effect, isolate the looping widget or use a separate animation chain for that state.

```dart
Row(
  children: [
    const Text('Syncing'),
    const SizedBox(width: 8),
    const Icon(Icons.sync),
  ],
)
    .animate(onPlay: (controller) => controller.repeat())
    .rotate(duration: 900.ms);
```

Use a state change to replace the syncing indicator with a completed icon instead of letting a repeating controller continue after the work is done. Also check `MediaQuery.of(context).disableAnimations` when motion is decorative; a loading status should still be understandable when animation is reduced.

## Practical checklist

- Put simultaneous effects in the same chain section.
- Add `then()` when the next effect must wait for completion.
- Use `delay` for a deliberate pause, not as a substitute for a missing state.
- Keep list staggering separate from each row’s local sequence.
- Repeat only an idle indicator, not the entire entrance animation.

The useful mental model is simple: direct effects describe one moment, `then()` moves the playhead, and `repeat()` decides whether that timeline starts again. Once those three decisions are explicit, multi-stage Flutter motion stays readable and does not need a web of controllers.
