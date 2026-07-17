---
layout: post
title: "flutter_animate Curves - Choose Motion That Fits the UI"
description: "Learn how flutter_animate applies curves per effect, with practical choices for entrances, looping motion, state feedback, and spring-like overshoot."
date: 2026-07-17
tags: [flutter_animate, animation, performance, accessibility]
comments: true
share: true
---

![Flutter animation curves shown as smooth, spring, and linear motion paths](/assets/images/flutter-animate-curves-motion.png)

`flutter_animate` makes the effect chain short, but the default curve is not automatically right for every widget. A 300ms `easeOut` entrance, a linear progress indicator, and an `elasticOut` success badge communicate three different things. Choosing the curve per effect is what keeps motion helpful instead of decorative noise.

## The curve belongs to the effect

One detail caused a surprising result in my first chained animation: I changed the curve on the second effect and expected the whole sequence to feel smoother. It only changed that effect's value mapping. Each effect owns its own `duration`, `delay`, and `curve`.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

Widget resultCard(bool saved) {
  return Card(
    child: ListTile(
      leading: const Icon(Icons.cloud_upload),
      title: Text(saved ? 'Saved' : 'Uploading'),
      trailing: Icon(saved ? Icons.check : Icons.sync),
    ),
  )
      .animate(key: ValueKey(saved))
      .fadeIn(duration: 180.ms, curve: Curves.easeOut)
      .slideY(
        begin: 0.08,
        end: 0,
        duration: 260.ms,
        curve: Curves.easeOutCubic,
      )
      .then(delay: 40.ms)
      .scaleXY(
        begin: 0.98,
        end: 1,
        duration: 140.ms,
        curve: Curves.easeOut,
      );
}
```

The `ValueKey` is intentional. It replays the small card transition when the upload state changes, while an unrelated parent rebuild leaves the animation alone. The `slideY` uses a stronger ease-out because the card should settle quickly. The final scale is subtle; a larger value makes a status update look like a button press.

## A practical curve guide

| UI situation | Good starting curve | Why |
| --- | --- | --- |
| New content entering | `easeOut` or `easeOutCubic` | Fast feedback, gentle landing |
| Content leaving | `easeIn` | The exit accelerates away from attention |
| Progress or scrubber | `linear` | Position remains honest over time |
| Small success emphasis | `easeOutBack` or short `elasticOut` | Adds a restrained overshoot |
| Infinite rotation | `linear` | Prevents a speed jump at the loop seam |

For a loading spinner, I use `Curves.linear` even when `easeInOut` looks nicer in a one-shot preview. Easing changes the angular speed, so the spinner slows down and speeds up on every repeat. That feels like a rendering bug rather than loading progress.

```dart
const Icon(Icons.sync)
    .animate(onPlay: (controller) => controller.repeat())
    .rotate(duration: 900.ms, curve: Curves.linear);
```

## When spring curves become too much

`elasticOut` and `bounceOut` are useful for a badge, check mark, or playful onboarding illustration. They are poor defaults for text, form validation, and navigation because the overshoot changes the apparent layout boundary. On a dense screen, a 1.0-to-1.12 scale can make nearby labels look like they are jumping.

I keep the overshoot local and short:

```dart
Icon(Icons.check_circle, color: Colors.green)
    .animate()
    .scaleXY(
      begin: 0.7,
      end: 1,
      duration: 320.ms,
      curve: Curves.elasticOut,
    )
    .fadeIn(duration: 120.ms, curve: Curves.easeOut);
```

If the same feedback appears repeatedly, switch to `easeOutBack` or a plain `easeOut`. A spring that is charming on the first save can become irritating after the tenth save.

## Three checks before shipping

- Keep entrance motion short enough that the user can act immediately. Around 160–300ms is a useful range for cards and labels.
- Use `linear` for motion that represents a value, time, or continuous gesture.
- Respect reduced-motion settings and skip overshoot when `MediaQuery.of(context).disableAnimations` is true.

The useful mental model is simple: `flutter_animate` supplies the timeline, but the curve supplies the meaning. Pick it based on whether the widget is entering, leaving, reporting progress, or asking for attention. Small distances and a deliberate curve usually outperform a longer chain full of effects.
