---
layout: post
title: "flutter_animate ValueAdapter - Drive Flutter Animations from Live Values"
description: "Use flutter_animate ValueAdapter to connect a Flutter slider or drag value to animation effects, with direction, smoothing, and rebuild pitfalls explained."
date: 2026-08-14
tags: [flutter_animate, animation, performance]
comments: true
share: true
---

![Flutter slider controlling an animated card with flutter_animate ValueAdapter](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

`flutter_animate` is usually introduced as a package for animations that play after a widget mounts. `ValueAdapter` is useful when the animation should not own the timeline. It lets an external value between `0` and `1` drive the effects, which is a much better fit for sliders, drag gestures, scroll progress, and scrubber controls.

## The problem with restarting an animation on every drag

Suppose a card should reveal itself as a user drags a slider. A common first attempt is to store an `AnimationController`, call `forward()` or `reverse()` in `onChanged`, and rebuild the card. That approach only expresses direction. It does not represent the actual position when the finger is halfway through the track.

`ValueAdapter` maps the external value to the complete `Animate` timeline:

```dart
class RevealDemo extends StatefulWidget {
  const RevealDemo({super.key});

  @override
  State<RevealDemo> createState() => _RevealDemoState();
}

class _RevealDemoState extends State<RevealDemo> {
  double _progress = 0.0;

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        Slider(
          value: _progress,
          onChanged: (value) => setState(() => _progress = value),
        ),
        Text(
          'Details revealed',
          style: Theme.of(context).textTheme.titleLarge,
        )
            .animate(
              adapter: ValueAdapter(_progress),
            )
            .fade(begin: 0.0, end: 1.0, duration: 300.ms)
            .slideX(begin: -0.25, end: 0.0, duration: 300.ms),
      ],
    );
  }
}
```

At `_progress == 0.0`, the text is transparent and shifted left. At `0.5`, both effects are halfway through their timelines. At `1.0`, the text is fully visible. There is no timer and no controller callback to synchronize.

The adapter is created from the current value, so rebuilding the `StatefulWidget` is intentional here. The `Animate` widget receives the new adapter value and updates its internal controller. Keep the range constrained to `0..1`; `Slider` already enforces that range, while a custom gesture or scroll calculation may need an explicit clamp.

## One value can coordinate several effects

The useful part is not just fading one child. Every effect in the chain reads from the same normalized timeline:

```dart
Card(
  child: const Padding(
    padding: EdgeInsets.all(20),
    child: Text('Order shipped'),
  ),
)
    .animate(adapter: ValueAdapter(_progress))
    .fade(begin: 0.2, end: 1.0)
    .scale(begin: const Offset(0.92, 0.92), end: const Offset(1, 1))
    .shimmer(delay: 120.ms, duration: 500.ms);
```

The effects still use their own durations and delays, but the adapter controls where the whole animation is sampled. This is different from chaining `then()`: `ThenEffect` schedules effects in time, while an adapter lets the user move through that schedule.

| Input source | Adapter pattern | Good use case |
|---|---|---|
| `Slider` or `PageController` calculation | `ValueAdapter(value)` | Preview and scrub controls |
| `ScrollController` | `ScrollAdapter` | Reveal content over a scroll range |
| `ValueNotifier<double>` | `ValueNotifierAdapter` | Shared progress without parent rebuilds |
| `AnimationController` | Regular `animate()` control | Time-based playback |

## Smoothing a value-driven animation

Direct updates are ideal for scrubbing because the UI follows the finger exactly. For a noisy source, such as a sensor or a rapidly changing calculation, pass `animated: true`:

```dart
final adapter = ValueAdapter(
  _progress,
  animated: true,
);

return Icon(Icons.wifi)
    .animate(adapter: adapter)
    .scale(begin: const Offset(0.8, 0.8), end: const Offset(1.2, 1.2));
```

With `animated: false`, the controller jumps to the new normalized value. With `animated: true`, the adapter animates toward the new value using a duration derived from the animation timeline. That feels nicer for discrete state changes, but it can make a drag control feel behind the pointer. I use direct updates for gestures and smoothing for values that arrive in steps.

## Restricting the direction

An adapter can also limit updates with `direction`. This matters for one-way progress indicators. If the source briefly moves backward because of a noisy measurement, a forward-only adapter can ignore that regression instead of replaying the entrance effect.

```dart
final adapter = ValueAdapter(
  uploadProgress,
  direction: Direction.forward,
);
```

Do not use this for a dismissible panel or a scrubber. Those interactions need both directions. A direction filter is a state rule, not an easing curve; it changes which values are accepted.

## Pitfalls I would check before shipping

First, do not treat `ValueAdapter` as a replacement for layout animation. A `MoveEffect` changes how the child is painted, but it does not make siblings recalculate their positions. Use a layout-aware Flutter widget when the surrounding layout must respond.

Second, normalize the input before constructing the adapter:

```dart
final progress = ((current - minValue) / (maxValue - minValue))
    .clamp(0.0, 1.0)
    .toDouble();
```

An out-of-range value can make the visual result look like a curve bug when the real problem is the input mapping. Also guard against `maxValue == minValue` before doing this calculation.

Finally, avoid creating a new `ValueNotifierAdapter` inside `build` if the notifier is meant to be shared. Create the notifier in `initState`, dispose it in `dispose`, and pass the same adapter lifecycle through the widget. For a plain slider value, the lightweight `ValueAdapter(_progress)` pattern is enough.

`ValueAdapter` turns `flutter_animate` from a play-once animation helper into a timeline that another part of the UI can control. Normalize the source, choose direct or animated updates based on the interaction, and use direction filtering only when backward motion is genuinely invalid.

- Use `ValueAdapter` for a live value in the `0..1` range.
- One adapter can scrub a complete chain of effects.
- Direct updates suit gestures; `animated: true` suits stepped values.
- Use `ScrollAdapter` or `ValueNotifierAdapter` when the input already has a dedicated lifecycle.

- [flutter_animate ValueAdapter API](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/ValueAdapter-class.html)
- [flutter_animate Adapter API](https://pub.dev/documentation/flutter_animate/latest/flutter_animate/Adapter-class.html)
