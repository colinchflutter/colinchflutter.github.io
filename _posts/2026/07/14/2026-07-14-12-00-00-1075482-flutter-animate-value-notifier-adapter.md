---
layout: post
title: "flutter_animate ValueNotifierAdapter - animate Flutter UI from state changes"
description: "Learn how to connect flutter_animate to a ValueNotifier for reusable state-driven animations, animated and instant updates, progress ranges, and disposal-safe Flutter code."
date: 2026-07-14
tags: [flutter_animate, animation, state_management, Flutter]
comments: true
share: true
---

![flutter_animate ValueNotifierAdapter state-driven animation](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

This image fits the topic because a notifier-driven animation is a small visual layer sitting on top of changing application state.

`flutter_animate`'s `ValueNotifierAdapter` is useful when a widget should follow a numeric state value instead of playing once when it enters the tree. A download progress ring, a reveal panel, and a slider-controlled preview all have the same shape: keep a value between `0.0` and `1.0`, then let the animation effects read that value.

I initially used `setState` plus several `Tween`s for this pattern. It worked, but every screen ended up owning slightly different animation plumbing. The adapter keeps the state source separate from the effect chain, which is the part that makes it reusable.

## The important contract: 0.0 to 1.0

`ValueNotifierAdapter` expects a `ValueNotifier<double>` whose value is normalized. It does not know that `0.75` means 75 percent downloaded or that it represents a selected tab. It only maps the number to the timeline of the `Animate` widget.

| State value | Animation position | Example visual state |
| ---: | ---: | --- |
| `0.0` | start | hidden or collapsed |
| `0.5` | middle | partly revealed |
| `1.0` | end | fully visible or expanded |

That normalization is worth doing at the boundary. Passing raw bytes, a score from 0 to 100, or an unbounded counter directly to the adapter produces clipped or permanently completed effects.

## A progress-controlled reveal

The following widget uses one notifier for both business progress and visual progress. The adapter drives a fade and a horizontal slide together.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';

class UploadPreview extends StatefulWidget {
  const UploadPreview({super.key});

  @override
  State<UploadPreview> createState() => _UploadPreviewState();
}

class _UploadPreviewState extends State<UploadPreview> {
  final progress = ValueNotifier<double>(0);

  @override
  void dispose() {
    progress.dispose();
    super.dispose();
  }

  void updateUpload(int sent, int total) {
    if (total <= 0) return;
    progress.value = (sent / total).clamp(0.0, 1.0);
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        LinearProgressIndicator(value: progress.value),
        const SizedBox(height: 16),
        const Text('Preview ready')
            .animate(adapter: ValueNotifierAdapter(progress))
            .fade(begin: 0, end: 1)
            .slideX(begin: -0.15, end: 0),
      ],
    );
  }
}
```

The `LinearProgressIndicator` in this small example does not rebuild when the notifier changes because it is not listening to the notifier. In a real screen, wrap that part in `ValueListenableBuilder`, or use a separate builder for the business-state display. The `Animate` widget attaches its own listener through the adapter, so the preview animation still responds.

That split is intentional. `ValueListenableBuilder` is for rebuilding Flutter widgets from state. `ValueNotifierAdapter` is for driving an animation timeline from the same state.

## Instant updates versus eased updates

The default adapter behavior follows the source value directly. If the notifier jumps from `0.2` to `0.9`, the effects jump to the corresponding point in the chain. That is usually right for a scrubber or a live drag, where the UI must not lag behind the finger.

For state that changes in discrete steps, set `animated: true`:

```dart
final statusProgress = ValueNotifier<double>(0);

Widget buildStatus() {
  return const Icon(Icons.cloud_done)
      .animate(
        adapter: ValueNotifierAdapter(
          statusProgress,
          animated: true,
        ),
      )
      .scaleXY(begin: 0.85, end: 1.0)
      .fadeIn();
}
```

With `animated: true`, a change from one value to another travels through the timeline instead of teleporting. This is a better fit for a status step such as queued → uploading → complete. It is a poor fit for a high-frequency pointer update because the animation can keep chasing values that have already changed.

The distinction is easy to miss:

| Source | Adapter mode | Why |
| --- | --- | --- |
| Drag or slider | `animated: false` | visual position should track input |
| Upload milestones | `animated: true` | transitions should feel deliberate |
| Timer progress | usually `false` | the source already changes smoothly |
| Boolean toggle mapped to `0`/`1` | `true` | avoid an abrupt visual jump |

## Avoid notifier lifetime bugs

The adapter subscribes to the notifier while `Animate` is mounted. The notifier must live at least as long as that widget. A common mistake is creating it inside `build`:

```dart
// Do not do this.
final progress = ValueNotifier<double>(0.0);
return Card().animate(
  adapter: ValueNotifierAdapter(progress),
);
```

Every rebuild creates a new source and resets the animation connection. Create it as a field in a `State`, or provide it from a state-management layer with a clearly owned lifetime. If the notifier belongs to a provider, the provider should dispose it when the feature is removed.

Also clamp external values before assigning them. Network callbacks can report a total of zero, retries can temporarily produce a value above one, and a late callback can arrive after the screen has started closing. Keeping the adapter input valid prevents confusing end-state behavior.

## When `target` is simpler

For a single local boolean, `target` is often less code:

```dart
AnimatedBuilder(
  animation: isExpanded,
  builder: (context, child) {
    return child!.animate(target: isExpanded.value ? 1 : 0).fade();
  },
  child: const Text('Details'),
)
```

Use `ValueNotifierAdapter` when the value itself has meaning across a continuous range, or when multiple widgets should share the same animation position. Use `target` when the widget only needs to animate between two semantic states and the transition is local.

The practical rule is simple: `target` describes a destination; `ValueNotifierAdapter` describes a position. Choosing the right one keeps state changes predictable and avoids an unnecessary `AnimationController`.

## Quick checklist

- Keep the notifier value in the `0.0`–`1.0` range.
- Use `animated: false` for direct manipulation and `true` for discrete state changes.
- Create the notifier outside `build`.
- Dispose a notifier that the widget owns.
- Use `ValueListenableBuilder` separately when non-animated widgets also need the value.
- Prefer `target` for a simple two-state transition.

`ValueNotifierAdapter` is small, but it fills a useful gap between a plain rebuild and a hand-written controller. The state remains numeric and testable, while `flutter_animate` handles how that state becomes motion.
