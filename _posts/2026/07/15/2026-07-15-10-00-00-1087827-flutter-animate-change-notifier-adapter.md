---
layout: post
title: "flutter_animate ChangeNotifierAdapter - animate Flutter UI from model state"
description: "Learn how to drive flutter_animate from a ChangeNotifier with normalized progress, smooth updates, direction filters, and safe disposal."
date: 2026-07-15
tags: [flutter_animate, animation, state_management, Flutter]
comments: true
share: true
---

![flutter_animate ChangeNotifierAdapter for state-driven Flutter animation](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

The useful part of `ChangeNotifierAdapter` is that a `ChangeNotifier` can drive a `flutter_animate` effect without turning every state update into a separate `AnimationController`. Give the adapter a notifier and a `valueGetter` that returns a number from `0.0` to `1.0`; the widget follows the model.

I reached for `setState` first when building a download card. That made the text update correctly, but the visual progress, opacity, and scale each ended up with their own timing code. `ChangeNotifierAdapter` keeps the state calculation in the model and the visual timeline in the widget.

## The adapter contract

The adapter listens for `notifyListeners()`. Whenever the notifier changes, it calls `valueGetter` and maps the returned value to the animation timeline. The getter is not a percentage formatter. It must return a normalized value.

| Model state | Getter result | Animation meaning |
| ---: | ---: | --- |
| 0 bytes loaded | `0.0` | effect starts |
| 500 of 1,000 bytes | `0.5` | effect is halfway through |
| 1,000 of 1,000 bytes | `1.0` | effect reaches its end |

That boundary is easy to miss when the model exposes raw byte counts. Keep the conversion in one place so the widget never has to know whether the source is bytes, pages, or steps.

## A notifier-backed download card

This small model exposes a ratio and clamps it so a late network callback cannot send an invalid value into the adapter.

```dart
import 'package:flutter/foundation.dart';

class DownloadProgress extends ChangeNotifier {
  int loaded = 0;
  int total = 1;

  double get ratio => (loaded / total).clamp(0.0, 1.0).toDouble();

  void update(int loadedBytes, int totalBytes) {
    loaded = loadedBytes;
    total = totalBytes <= 0 ? 1 : totalBytes;
    notifyListeners();
  }
}
```

The widget supplies the model to `ChangeNotifierAdapter` and composes ordinary effects on top of it. `animated: true` smooths jumps between values instead of snapping directly to the latest progress.

```dart
class DownloadCard extends StatefulWidget {
  const DownloadCard({super.key});

  @override
  State<DownloadCard> createState() => _DownloadCardState();
}

class _DownloadCardState extends State<DownloadCard> {
  final _progress = DownloadProgress();
  late final ChangeNotifierAdapter _adapter;

  @override
  void initState() {
    super.initState();
    _adapter = ChangeNotifierAdapter(
      _progress,
      () => _progress.ratio,
      animated: true,
    );
  }

  @override
  void dispose() {
    _progress.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: _progress,
      builder: (context, _) => Column(
        children: [
          CircularProgressIndicator(value: _progress.ratio),
          const SizedBox(height: 12),
          const Text('Downloading…')
              .animate(adapter: _adapter)
              .fade(begin: 0.45, end: 1.0)
              .scale(begin: const Offset(0.96, 0.96)),
        ],
      ),
    );
  }
}
```

The `CircularProgressIndicator` above is deliberately bound to the same model, so its numeric value and the animated label cannot drift apart. In a real screen, the download service would call `_progress.update(loaded, total)` as chunks arrive.

## `animated` changes the feel

The default adapter behavior updates the animation position immediately. That is a good fit for a scrubber or a value that must stay exactly under the user's finger. A download callback is different: network chunks can arrive in uneven bursts, and a snap from `0.42` to `0.58` looks noisy.

| Setting | Best fit | Trade-off |
| --- | --- | --- |
| `animated: false` | slider, drag, exact state reflection | can look abrupt |
| `animated: true` | progress, counters, remote state | can visually lag during rapid updates |

For a progress indicator, I use `animated: true` and keep updates reasonably frequent. If the source emits dozens of values per frame, smoothing every one is unnecessary. Throttle the source or publish meaningful changes such as every 1–2 percent.

## Filtering updates with `direction`

Some effects should only move forward. A completion badge should not fade out just because a retry resets the download to `0.0`. Set `direction: Direction.forward` when decreases should be ignored by this animation.

```dart
const Icon(Icons.check).animate(
  adapter: ChangeNotifierAdapter(
    _progress,
    () => _progress.ratio,
    animated: true,
    direction: Direction.forward,
  ),
).fadeIn().scale();
```

Use this selectively. A retry UI usually needs to show the reverse transition, while a one-way “completed” affordance may not. The direction filter changes the animation listener, not the underlying model, so other widgets still see the reset.

## Traps that caused confusing results

First, do not return a raw integer or a value above `1.0`. `ChangeNotifierAdapter` is driving an animation timeline, not a `ProgressIndicator` API. Normalize and clamp inside the model.

Second, do not create the notifier inside `build`. Every rebuild would create a different source and could attach a new adapter listener. Keep it as a field, or inject a longer-lived notifier from a provider.

Third, dispose the notifier at the same ownership boundary that created it. The adapter detaches from the notifier with the `Animate` widget, but it does not mean a notifier owned by your `State` should live forever.

Finally, use `ValueNotifierAdapter` when the source is already a single `ValueNotifier<double>`. `ChangeNotifierAdapter` is the better fit when the ratio is derived from multiple fields such as `loaded`, `total`, and a retry state.

## Practical rule

Use `ChangeNotifierAdapter` when the animation represents model progress rather than a one-time entrance. Normalize the getter, decide whether updates should snap or interpolate, and make direction filtering a per-effect decision. That gives one source of truth for state while keeping the animation chain declarative.

The important relationship is simple: the notifier owns the value, the getter normalizes it, and `flutter_animate` owns the visual response.

References:

- [ChangeNotifierAdapter API](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/ChangeNotifierAdapter-class.html)
- [flutter_animate Adapter API](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/Adapter-class.html)
