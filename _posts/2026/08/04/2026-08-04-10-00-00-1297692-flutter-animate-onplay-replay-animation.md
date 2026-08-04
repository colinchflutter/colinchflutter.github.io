---
layout: post
title: "flutter_animate onPlay - Replay Flutter Animations After Async State Changes"
description: "Use flutter_animate onPlay and autoPlay to replay Flutter entrance animations after async updates or route returns without duplicate timers."
date: 2026-08-04
tags: [flutter_animate, animation, state_management, Flutter]
comments: true
share: true
---

![Flutter animation timeline for replaying a flutter_animate sequence](/assets/images/flutter-animate-curves-motion.png)

*The important detail is that replaying a sequence should reset its controller, not create another delayed animation beside it.*

`flutter_animate` plays an entrance sequence when an `Animate` widget is inserted. That is perfect for a first load, but it is not enough when a screen receives new data, returns from another route, or needs a deliberate “try again” animation. The `onPlay` callback gives access to the package's controller at the moment playback starts, so a parent can request a clean replay without adding a `Timer` or manually rebuilding the whole screen.

## A replayable async card

Keep `autoPlay: false` when the animation should wait for a real event. The controller is then driven by a button, a completed request, or a route lifecycle callback.

```dart
class ResultCard extends StatefulWidget {
  const ResultCard({super.key});

  @override
  State<ResultCard> createState() => _ResultCardState();
}

class _ResultCardState extends State<ResultCard>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller = AnimationController(
    vsync: this,
    duration: const Duration(milliseconds: 360),
  );
  bool _loading = false;

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  Future<void> _refresh() async {
    setState(() => _loading = true);
    await Future<void>.delayed(const Duration(milliseconds: 700));
    if (!mounted) return;

    setState(() => _loading = false);
    _controller.forward(from: 0);
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Animate(
          autoPlay: false,
          controller: _controller,
          onPlay: (controller) => controller.forward(from: 0),
          effects: const [
            FadeEffect(duration: Duration(milliseconds: 260)),
            SlideEffect(
              begin: Offset(0, .12),
              end: Offset.zero,
              duration: Duration(milliseconds: 360),
            ),
          ],
          child: Card(
            child: ListTile(
              title: Text(_loading ? 'Refreshing…' : 'Latest result'),
              subtitle: const Text('The card re-enters after new data arrives.'),
            ),
          ),
        ),
        FilledButton(onPressed: _loading ? null : _refresh, child: const Text('Refresh')),
      ],
    );
  }
}
```

The external controller is intentional here. It gives the async event a stable way to reach the animation while the card remains mounted. `forward(from: 0)` resets and starts the sequence in one call. Without the `from: 0`, a completed animation can appear to do nothing because its controller is already at the end.

## `onPlay` is not a trigger

`onPlay` runs when playback begins. It configures the controller, but it does not itself decide when playback begins. That distinction matters when the trigger comes from a `FutureBuilder`, a Riverpod provider, or a route observer. Call `forward(from: 0)` from the event that owns the state transition, and use `onPlay` for shared setup such as speed or repeat behavior.

```dart
Animate(
  autoPlay: false,
  onPlay: (controller) {
    controller.duration = const Duration(milliseconds: 500);
  },
  effects: const [ScaleEffect(), FadeEffect()],
  child: const Icon(Icons.cloud_done),
)
```

For a looping loading indicator, `controller.repeat()` belongs in `onPlay`, but do not use it for a one-shot refresh animation. A repeat callback never reaches a settled state and can keep a screen animating after its data has already arrived. Stop it explicitly with `controller.stop()` or use a separate loading widget.

## The traps that caused the bug

| Situation | Symptom | Better choice |
|---|---|---|
| Rebuild after `setState` | Animation unexpectedly replays | Keep a stable key and trigger the controller explicitly |
| Call `forward()` twice quickly | Janky or skipped visual state | Guard the event while loading or stop before restarting |
| Use `Future.delayed` for replay | Old timer animates stale data | Start playback inside the async completion path |
| Put `repeat()` on a result card | Battery and ticker usage continue | Reserve repeat for an actual loading state |

The practical rule is simple: let the state change own the trigger, and let `flutter_animate` own the visual timeline. `autoPlay: false`, an external controller, and `forward(from: 0)` cover most replay cases without timer bookkeeping.

- [flutter_animate on pub.dev](https://pub.dev/packages/flutter_animate)
- [flutter_animate API documentation](https://pub.dev/documentation/flutter_animate/latest/)
